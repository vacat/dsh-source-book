# 第 2 章 环境与依赖

pi-durable 是 pi monorepo 里的一个包，不要把它从父仓库剥出来单独读。所有直接依赖（chord、pi-ai）都在同一棵 workspace 树里。本章代替"README 上手教程"，回答两个问题：**读代码前装什么**，和**跑测试怎么跑**。

## 2.1 依赖与版本基线

`packages/durable/package.json` 上游版本基线（阅读时以你手上的仓库为准）：

- 硬依赖：
  - `@earendil-works/chord ^1.0.3`：文档状态管理与 delta 计算的基础设施。
  - `@earendil-works/pi-ai ^1.0.3`：模型抽象层，所有 `Model` / `Message` / `Tool` 类型从这里来。
  - `diff 8.0.4`、`typebox 1.3.27`：diff 与 Schema 校验。
- 本仓库根 `package.json` 的 devDependencies：`shx`、`vitest`。

`sideEffects: false` 表明 tree-shaking 是一等公民——pi-durable 的工具、`session`、`harness` 等入口独立编译，不会整个包一起拖进来。

## 2.2 目录地图

```
packages/durable/
├── src/
│   ├── session/          Session 串行 commit 与事务
│   │   ├── session.ts        SessionImpl 管线
│   │   ├── transaction.ts    Tx、prepare/commit 语义
│   │   ├── observation.ts    提交后可观察状态
│   │   └── forks.ts          fork 语义
│   ├── storage/
│   │   ├── memory.ts         MemoryStorage
│   │   ├── sqlite/           SQLite（含 migrations）
│   │   └── jsonl/            JSONL 追加日志
│   ├── harness/
│   │   ├── harness.ts        Harness.open()
│   │   ├── scheduler.ts      任务调度
│   │   ├── generation.ts     模型回合任务
│   │   ├── tool.ts           工具调用任务
│   │   └── ...               conversations / compaction / usage 等
│   ├── tools/            read/write/edit/bash 工具
│   ├── env/              ExecutionEnv / NodeExecutionEnv
│   ├── tasks.ts          defineTask()
│   ├── documents.ts      defineDoc()
│   └── types.ts          全部类型定义
├── test/
│   ├── examples/         32 个可运行示例
│   └── *.test.ts         47 个单元测试
└── docs/spec.md          规范（normative）
```

## 2.3 单仓库视角的快速上手

从本仓库根目录出发（`packages/durable` 下用 pnpm workspace）：

```bash
# 在 pi 仓库根目录
pnpm install

# 只跑 pi-durable 的单元测试
pnpm --dir packages/durable test

# 跑某个具体文件
pnpm --dir packages/durable exec vitest run test/harness-compaction.test.ts

# 从源码跑一个示例
node --conditions=source --experimental-strip-types packages/durable/test/examples/26-coding-agent.ts
```

前两点是普通 `pnpm` 命令；最后一个是 Node 的原生 TS 支持（`--experimental-strip-types`）。在生产运行或 SDK 里，pi-durable 会先被 tsc 编译到 `dist/` 再以 `@earendil-works/pi-durable` 的常规 ESM 包加载。

## 2.4 读源码前的预期

pi-durable 的定位（README 原话）："A durable agent harness. Conversations, model turns, tool calls, and your own state are committed to storage before anything is shown."——**它不是 UI，不是 API server，也不是 prompt-assembler**。它的 API 面窄到只有 `Harness` 和一个 `Conversation` 句柄；`Agent`（`@earendil-works/pi-agent-core`）提供你实际的模型循环。

理解清楚"pi-durable 只是把转移状态托底"，你后续读 `session/` 时能少陷很多坑。最贴切的对应比喻（不要过度引申）：Agent 是发动机，pi-durable 是把发动机的曲轴箱封起来，让停机时你能从曲轴箱的连续纸质记录恢复出停机前的气息。
