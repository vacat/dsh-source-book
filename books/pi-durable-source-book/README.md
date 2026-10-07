# pi-durable 源码进阶：读懂可持久化 Agent Harness

> 面向已掌握 Agent 基础概念的工程师，深入 pi-durable 源码，逐行读懂一个"可持久化 Agent Harness"的工程实现。

本书是《Agent 技术实战》《dsh 源码进阶》《pi 的设计艺术》同一系列的姊妹篇，专门拆 pi 生态里的 `@earendil-works/pi-durable`。核心回答一个问题：**Agent 会话如何做到"先落盘、再展示、崩溃后原样续上"**。

- 在线阅读：<https://vacat.github.io/pi-durable-source-book/>
- 参考源码：[earendil-works/pi/packages/durable](https://github.com/earendil-works/pi/tree/main/packages/durable)

## 内容

- **第一部分 总览与地基**：pi-durable 在 pi 生态的位置、四层架构、依赖与构建。
- **第二部分 运行时深读**：Session 串行事务管线、Memory/SQLite/JSONL 三后端、Harness 语义、工具执行与重放。
- **第三部分 扩展与工程面**：conformance 测试套件、任务状态机与 Storage 契约、扩展 / Hook / 工具 DSL。
- **附录**：快速导出索引与学习清单。

## 本地构建

```bash
brew install mdbook
cargo install mdbook-mermaid
mdbook build        # 输出 book/
mdbook serve --open # http://localhost:3000
```

要求 mdBook >= 0.5 与 mdbook-mermaid 0.17。构建依赖与源码版本基线见 `src/preface.md`。

## 许可

本书内容以 MIT 许可发布，与参考源码仓库保持一致。
