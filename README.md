# Project-Tuning-Notes
Project Issue & Tuning Log

## 📚 文档查看工具

### 在线访问
- **在线地址**: [https://d32c85f8.pinit.eth.limo/](https://d32c85f8.pinit.eth.limo/)

### 本地使用
- 打开 `tools/view_documentation.html` 文件，在浏览器中加载并渲染 Markdown 文档
- 支持 Markdown 文档和 PlantUML 图表渲染

## 📑 文档目录

### 性能优化分析

#### [getAllTableAndFieldInfo 并行流优化性能分析](./markdown/parallel-stream-optimization-analysis.md)
- **问题背景**: Feign 调用接口响应时间长达 6 分钟以上，优化后可在 10 秒内完成
- **优化方案**: 使用并行流（parallelStream）结合客户端复用
- **性能提升**: 约 17-20 倍，从 6 分钟降至 10 秒内
- **关键优化点**:
  - 数据库查询次数：从 N 次降为 1 次
  - 客户端获取次数：从 N 次降为 1 次
  - 远程查询执行方式：从串行变为并行
- **包含内容**: 原始代码分析、优化方案对比、UML 时序图、性能对比数据、优化策略总结

---

*本文档目录将持续更新和完善*
