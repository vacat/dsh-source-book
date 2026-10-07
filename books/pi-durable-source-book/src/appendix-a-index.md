# 附录 A 快速索引

pi-durable 的"公共 API"不是从 `endpoint` 一栏读出的——它是一组"语义如果一致, 任何后端实现都可以替换"的导出物。本附录把它们按**读码出口**和**去哪里看**分列, 当你需要从这一页查一个东西时不必从第 1 章翻回。

## 主要导出（`from "@earendil-works/pi-durable"`）

| 导出 | 用途 | 源码 |
|---|---|---|
| `Harness` | 打开存储、装配调度器, 提供 `root()` / `createConversation()` / `fork()` | `src/harness/harness.ts` |
| `createSession` | 直接使用 Session 层事务, 绕开 Harness 业务语义 | `src/session/session.ts` |
| `MemoryStorage` | in-memory 后端; 单测和本地调试 | `src/storage/memory.ts` |
| `defineTask` | 定义一个自定义的可恢复状态机 | `src/tasks.ts` |
| `defineDoc` | 定义一个由 Chord 追踪的结构化文档 | `src/documents.ts` |
| `defineTool` | 定义一个可执行的 Agent 工具（含 TypeBox 参数检查） | `src/harness/define.ts` |
| `defineExtension` / `hook` / `section` / `wrapTool` / `wrapSection` | 打包工具、提示词段、钩子、装饰器为一个可安装的插件 | `src/harness/define.ts` |
| `AssistantEntry` / `UserEntry` / `SystemEntry` / `ToolResultEntry` / `CompactionEntry` / `ResetEntry` | 对话记录的可写入口 | `src/entries.ts` |
| `configure` | 对一次 `tx` 应用 `AgentChange` | `src/harness/agent.ts` |

## 常用子路径

| 子路径 | 用途 |
|---|---|
| `@earendil-works/pi-durable/env` | 环境接口抽象（`ExecutionEnv`, `FileSystem`, `Shell`） |
| `@earendil-works/pi-durable/env/node` | Node 实现 `NodeExecutionEnv` |
| `@earendil-works/pi-durable/tools` | 内置 coding 工具（read / write / edit / bash）与 `CodingTools` 扩展 |
| `@earendil-works/pi-durable/storage/sqlite/node` | Node 上的 SQLite 后端入口 |
| `@earendil-works/pi-durable/storage/jsonl/node` | Node 上的 JSONL 后端入口 |
| `@earendil-works/pi-durable/storage/memory` | 纯内存后端 |
| `@earendil-works/pi-durable/testing` | `registerStorageConformance()`, `registerEnvConformance()`, 断言与基准 |

## docs 目录的三个文件

- `docs/spec.md`——**权威规范** (`normative`)：记录 Session / Conversation / Entry / Task / Document / Storage / API footguns 等完整语义。
- `docs/pico-v5-handoff.md`——把 spec 变成代码的一揽子实施计划（23 个 package 的 milestone 序列）。
- `docs/pico-v5-chord-usage.md`——pi-durable 如何使用 chord 的 delta 追踪体系。

## 三个后端怎么选

| 场景 | 建议 |
|---|---|
| 单测、单机 | `MemoryStorage` |
| 桌面 / CLI 持久化 | `openNodeSqliteStorage()` |
| 追加日志型 / 不希望依赖原生存原生依赖 | `openNodeJsonlStorage()`（可传 `{ fsync: true }` 收紧提交点） |
| 云函数 / 自定义后端 | 修复 `Storage` 接口 + 用 `registerStorageConformance()` 校验 |
