# getAllTableAndFieldInfo 并行流优化性能分析

## 1. 问题背景

在 `DatasourceServiceImpl.getAllTableAndFieldInfo` 方法中，通过 Feign 调用该接口时存在严重的性能问题，原始实现导致接口响应时间长达 **6分钟以上**，优化后可在 **10秒内** 完成查询。

## 2. 原始代码分析

### 2.1 原始代码

```java
@Override
public List<Map<String, Object>> getAllTableAndFieldInfo(Long datasourceId) {
    List<String> tables = showTables(datasourceId);
    if (CollectionUtil.isNotEmpty(tables)) {
        return tables.stream().filter(StringUtil::isNotBlank).map(table -> {
            List<FieldInfo> fieldInfos = getFieldInfoByTable(datasourceId, table);
            Map<String, Object> map = new HashMap<>();
            map.put("tableName", table);
            map.put("fields", fieldInfos);
            return map;
        }).collect(Collectors.toList());
    }
    return null;
}
```

### 2.2 getFieldInfoByTable 方法

```java
@Override
@SneakyThrows
public List<FieldInfo> getFieldInfoByTable(Long datasourceId, String tableName) {
    Datasource datasource = baseMapper.selectById(datasourceId);  // 数据库查询
    BaseDatasourceDTO baseDatasourceDTO = DatasourceWrapper.build().entityVO(datasource);
    DataSourceClient client = DataSourceClientProvider.getInstance().getClient(baseDatasourceDTO);
    return client.getFieldInfoByTable(tableName);  // 远程数据源查询
}
```

### 2.3 问题识别

原始代码存在以下性能瓶颈：

| 问题点 | 描述 | 影响 |
|--------|------|------|
| 串行执行 | 使用 `stream()` 串行处理所有表 | 总时间 = N × 单表查询时间 |
| 重复数据库查询 | 每次循环都执行 `selectById` | N 次额外数据库查询 |
| 重复获取客户端 | 每次循环都获取 `DataSourceClient` | N 次客户端创建/获取开销 |

## 3. 优化方案对比

### 3.1 优化后代码（并行流 + 客户端复用）

```java
@Override
@SneakyThrows
public List<Map<String, Object>> getAllTableAndFieldInfo(Long datasourceId) {
    List<String> tables = showTables(datasourceId);
    if (CollectionUtil.isNotEmpty(tables)) {
        // 提前获取数据源和客户端，避免循环内重复查询
        Datasource datasource = baseMapper.selectById(datasourceId);
        BaseDatasourceDTO baseDatasourceDTO = DatasourceWrapper.build().entityVO(datasource);
        DataSourceClient client = DataSourceClientProvider.getInstance().getClient(baseDatasourceDTO);
        
        // 使用并行流加速
        return tables.parallelStream().filter(StringUtil::isNotBlank).map(table -> {
            List<FieldInfo> fieldInfos = client.getFieldInfoByTable(table);
            Map<String, Object> map = new HashMap<>();
            map.put("tableName", table);
            map.put("fields", fieldInfos);
            return map;
        }).collect(Collectors.toList());
    }
    return null;
}
```

## 4. 执行流程对比（UML 时序图）

### 4.1 原始代码执行流程

```plantuml
@startuml original-sequence
title 原始代码执行流程（串行）

actor Client as C
participant "DatasourceServiceImpl" as DSI
participant "BaseMapper" as BM
participant "DataSourceClientProvider" as DSCP
participant "DataSourceClient" as DSC
database "本地数据库" as LDB
database "远程数据源" as RDB

C -> DSI: getAllTableAndFieldInfo(datasourceId)
activate DSI

DSI -> DSI: showTables(datasourceId)
DSI <-- DSI: tables[N个表]

loop 对于每个表（串行执行）
    DSI -> DSI: getFieldInfoByTable(datasourceId, table)
    activate DSI #LightBlue
    
    DSI -> BM: selectById(datasourceId)
    BM -> LDB: SELECT * FROM datasource
    LDB --> BM: datasource
    BM --> DSI: datasource
    
    DSI -> DSCP: getClient(baseDatasourceDTO)
    DSCP --> DSI: client
    
    DSI -> DSC: getFieldInfoByTable(tableName)
    DSC -> RDB: 查询字段元数据
    note right: 网络IO等待\n（约3-5秒/表）
    RDB --> DSC: fieldInfos
    DSC --> DSI: fieldInfos
    
    deactivate DSI
end

DSI --> C: List<Map<String, Object>>
deactivate DSI

note over DSI: 总耗时 = N × (DB查询 + 客户端获取 + 远程查询)\n假设N=100, 每次3.6秒 → 总计360秒(6分钟)
@enduml
```

### 4.2 优化后代码执行流程

```plantuml
@startuml optimized-sequence
title 优化后代码执行流程（并行）

actor Client as C
participant "DatasourceServiceImpl" as DSI
participant "BaseMapper" as BM
participant "DataSourceClientProvider" as DSCP
participant "DataSourceClient" as DSC
participant "ForkJoinPool" as FJP
database "本地数据库" as LDB
database "远程数据源" as RDB

C -> DSI: getAllTableAndFieldInfo(datasourceId)
activate DSI

DSI -> DSI: showTables(datasourceId)
DSI <-- DSI: tables[N个表]

DSI -> BM: selectById(datasourceId)
note right: 仅查询1次
BM -> LDB: SELECT * FROM datasource
LDB --> BM: datasource
BM --> DSI: datasource

DSI -> DSCP: getClient(baseDatasourceDTO)
note right: 仅获取1次
DSCP --> DSI: client (复用)

DSI -> FJP: parallelStream().map(...)
activate FJP

note over FJP: 并行执行（多个Worker线程同时工作）

FJP -> DSC: getFieldInfoByTable(table1)
activate DSC #LightBlue
DSC -> RDB: 查询字段元数据
RDB --> DSC: fieldInfos1
DSC --> FJP: result1
deactivate DSC

FJP -> DSC: getFieldInfoByTable(table2)
activate DSC #LightGreen
DSC -> RDB: 查询字段元数据
RDB --> DSC: fieldInfos2
DSC --> FJP: result2
deactivate DSC

FJP -> DSC: getFieldInfoByTable(tableN)
activate DSC #LightCoral
DSC -> RDB: 查询字段元数据
RDB --> DSC: fieldInfosN
DSC --> FJP: resultN
deactivate DSC

note over FJP: 上述调用实际并行执行\n而非顺序执行

FJP --> DSI: 聚合所有结果
deactivate FJP

DSI --> C: List<Map<String, Object>>
deactivate DSI

note over DSI: 总耗时 ≈ N / 并行度 × 单次远程查询时间\n假设N=100, 并行度=16, 每次3秒 → 总计约19秒
@enduml
```

## 5. 性能提升原理分析

### 5.1 并行度计算

```plantuml
@startuml parallelism-calculation
title Java ForkJoinPool 默认并行度

start

:获取 CPU 核心数;
note right: Runtime.getRuntime().availableProcessors()

:计算默认并行度;
note right: parallelism = CPU核心数 - 1\n（保留1个核心给主线程）

if (CPU = 8核) then (yes)
    :并行度 = 7;
else if (CPU = 16核) then (yes)
    :并行度 = 15;
else (其他)
    :并行度 = N - 1;
endif

:ForkJoinPool.commonPool();
note right: 所有 parallelStream 共享此线程池

stop
@enduml
```

### 5.2 时间复杂度对比

```plantuml
@startuml time-complexity
title 时间复杂度对比

rectangle "原始串行方案" as Original {
    card "表1查询\n3.6秒" as T1
    card "表2查询\n3.6秒" as T2
    card "...\n..." as TDots
    card "表N查询\n3.6秒" as TN
}

T1 --> T2: 等待完成
T2 --> TDots: 等待完成
TDots --> TN: 等待完成

rectangle "优化并行方案" as Optimized {
    rectangle "时间片1" as Slot1 {
        card "表1" as P1
        card "表2" as P2
        card "表..." as P3
        card "表16" as P4
    }
    rectangle "时间片2" as Slot2 {
        card "表17" as P5
        card "表18" as P6
        card "表..." as P7
        card "表32" as P8
    }
    rectangle "时间片N/16" as SlotN {
        card "表N-15" as PN1
        card "..." as PN2
        card "表N" as PN3
    }
}

Slot1 --> Slot2: 批次完成
Slot2 --> SlotN: 批次完成

note bottom of Original
  总时间 = N × T
  100表 × 3.6秒 = 360秒
end note

note bottom of Optimized
  总时间 ≈ (N / P) × T
  (100 / 16) × 3秒 ≈ 19秒
end note
@enduml
```

## 6. 性能对比数据

### 6.1 理论计算

假设条件：
- 数据源有 **100** 个表
- 每次 `getFieldInfoByTable` 调用耗时 **3.6秒**（包含网络IO）
- CPU 为 **16核**，ForkJoinPool 并行度为 **15**
- 本地数据库查询 `selectById` 耗时 **10ms**

| 指标 | 原始方案 | 优化方案 |
|------|----------|----------|
| 数据库查询次数 | 100次 | 1次 |
| 客户端获取次数 | 100次 | 1次 |
| 远程查询执行方式 | 串行 | 并行(15线程) |
| 理论总耗时 | 100 × 3.6s = **360秒** | 1s + (100/15) × 3s ≈ **21秒** |
| 性能提升 | 基准 | **约17倍** |

### 6.2 实际场景分析

```plantuml
@startuml performance-chart
title 性能提升对比（假设100个表）

rectangle "原始方案耗时分解" as OriginalBreakdown {
    card "DB查询: 100 × 10ms = 1s" as ODB
    card "客户端获取: 100 × 5ms = 0.5s" as OClient
    card "远程查询: 100 × 3.5s = 350s" as ORemote
}

rectangle "优化方案耗时分解" as OptimizedBreakdown {
    card "DB查询: 1 × 10ms = 0.01s" as NDB
    card "客户端获取: 1 × 5ms = 0.005s" as NClient
    card "并行远程查询: 7批 × 3.5s = 24.5s" as NRemote
}

note bottom of OriginalBreakdown
  总计: 约 351.5秒 ≈ 6分钟
end note

note bottom of OptimizedBreakdown
  总计: 约 24.5秒 < 30秒
end note
@enduml
```

## 7. 关键优化点总结

### 7.1 组件交互关系

```plantuml
@startuml component-diagram
title 组件依赖关系

package "DatasourceService" {
    [getAllTableAndFieldInfo] as Main
    [showTables] as ShowTables
    [getFieldInfoByTable] as GetField
}

package "Infrastructure" {
    [BaseMapper] as Mapper
    [DataSourceClientProvider] as Provider
    database "本地PostgreSQL" as LocalDB
}

package "Plugin" {
    [DataSourceClient] as Client
    database "远程数据源\n(MySQL/Oracle/...)" as RemoteDB
}

package "Java Concurrent" {
    [ForkJoinPool] as FJP
    [parallelStream] as PS
}

Main --> ShowTables: 1. 获取表列表
Main --> Mapper: 2. 查询数据源配置\n(优化: 仅1次)
Mapper --> LocalDB
Main --> Provider: 3. 获取客户端\n(优化: 仅1次)
Main --> PS: 4. 并行处理
PS --> FJP: 任务分发
FJP --> Client: 5. 并行查询字段
Client --> RemoteDB: 6. 元数据查询

note right of FJP
  默认使用 commonPool
  并行度 = CPU核心数 - 1
end note
@enduml
```

### 7.2 优化策略总结

```plantuml
@startuml optimization-strategy
title 优化策略类图

class OriginalImplementation {
    - tables: List<String>
    + getAllTableAndFieldInfo(datasourceId): List
    --
    问题1: 循环内重复查询数据源
    问题2: 循环内重复获取客户端
    问题3: 串行执行所有表查询
}

class OptimizedImplementation {
    - tables: List<String>
    - client: DataSourceClient
    + getAllTableAndFieldInfo(datasourceId): List
    --
    优化1: 数据源仅查询1次
    优化2: 客户端仅获取1次并复用
    优化3: parallelStream并行执行
}

class PerformanceGain {
    + dbQueryReduction: "100次 → 1次"
    + clientCreationReduction: "100次 → 1次"
    + executionMode: "串行 → 并行"
    + speedup: "约17倍"
}

OriginalImplementation --> OptimizedImplementation: 重构
OptimizedImplementation --> PerformanceGain: 带来
@enduml
```

## 8. 注意事项

### 8.1 并行流使用注意

1. **线程安全**: `DataSourceClient` 必须是线程安全的，或者使用连接池
2. **资源竞争**: 并行度过高可能导致远程数据源连接池耗尽
3. **异常处理**: 并行流中的异常会被包装为 `CompletionException`
4. **结果顺序**: `parallelStream` 不保证处理顺序，但 `collect` 会保持原顺序

### 8.2 进一步优化建议

```plantuml
@startuml further-optimization
title 进一步优化方向

start

:当前优化\n(并行流 + 客户端复用);

fork
    :方向1: 使用批量查询API;
    note right
        DataSourceClient.getFieldInfoByTables()
        一次SQL查询所有表字段
    end note
fork again
    :方向2: 添加本地缓存;
    note right
        使用 Caffeine/Redis 缓存
        减少重复查询
    end note
fork again
    :方向3: 自定义线程池;
    note right
        不使用 ForkJoinPool.commonPool()
        避免影响其他并行流操作
    end note
end fork

:综合优化方案;

stop
@enduml
```

## 9. 结论

通过将 `stream()` 改为 `parallelStream()` 并结合客户端复用优化，可以实现：

| 优化项 | 效果 |
|--------|------|
| 数据库查询次数 | 从 N 次降为 1 次 |
| 客户端获取次数 | 从 N 次降为 1 次 |
| 远程查询执行方式 | 从串行变为并行 |
| 总体性能提升 | **约 17-20 倍** |
| 实际耗时 | 从 6分钟 降至 10秒内 |

这种优化方式改动量小（仅需修改几行代码），但效果显著，是处理批量 IO 密集型任务的经典优化模式。
