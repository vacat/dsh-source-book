# 第 7 章 测试与工具层基础设施

pi-durable 的 testing 支持有三块: **conformance 套件**（保证自定义后端和自定义环境在语义上与内置后端等价）、**examples**（可运行参考实现）和**单元测试树**（47 个 .test.ts）。本章讲解前两块怎么读、怎么运行、能支撑哪类心智模型的迁移。

## 7.1 Conformance 套件的定位

`packages/durable/src/testing/*` 的职责不是“测 pi-durable 本身”，而是**给外部的自定义实现提供验收标准**:

- `storage-conformance.ts`（57 KB，全仓最大的单文件是它）——自定义 `Storage` 后端的接口一致性;
- `env-conformance.ts`——自定义 `ExecutionEnv` 的接口一致性;
- `storage-benchmark.ts`——衡量后端在常见写/读操作上的基准。

写法（来自 README）:

```typescript
import { registerStorageConformance } from "@earendil-works/pi-durable/testing";
import { describe, expect, it } from "vitest";

registerStorageConformance({ describe, expect, it }, "My Storage", async (use) => {
  const storage = await openMyStorage();
  try {
    await use(storage);
  } finally {
    await closeMyStorage(storage);
  }
});
```

### 后端一致性测试的关键是什么?

通常要用这四个类测试来证明三个词:

1. **提交性**: 所有 commit 成功后的读可见, commit 失败的写不可见。
2. **单调性**: 自增 ID 不重复, `Seq` 不回退。
3. **灾难恢复**: 重启后能恢复所有已提交状态。
4. **并发**: 多个提交的可见序与提交序一致。

## 7.2 单元测试树

47 个测试文件, 很大一部分名字对得上 README 的公开能力:

- `sqlite-storage.test.ts`、`jsonl-storage.test.ts` — 对两后端各自完整测试;
- `session-*.test.ts` — 事务、文档、提交、观察者（9 个文件）;
- `harness-*.test.ts` — 从 `harness-tasks` 到 `harness-structured` 的一切运行时语义;
- `tools-*.ts` — 内置工具各通道;
- `env-node.test.ts`（约 1,000 行）— `NodeExecutionEnv` 的文件系统能力。

跑法（自仓库根目录）:

```bash
pnpm --dir packages/durable test
pnpm --dir packages/durable exec vitest run test/harness-compaction.test.ts
```

## 7.3 Examples: 一棵被维护起来的“参考实现树”

`test/examples/` 有 32 个编号目录, 每个都是一段**可直接运行**的脚本, 成对覆盖 pi-durable 的全部能力。README 的 Examples 一节写得非常清楚, 摘录对应:

```bash
# 例子: 从最简聊天到一日式 coding agent
node --conditions=source --experimental-strip-types test/examples/14-chat.ts
node --conditions=source --experimental-strip-types test/examples/26-coding-agent.ts
```

## 7.4 运行测试要一套 node 端的环境

当你写扩展或自定义 backend 时, pi-durable 所期望的 Node 版本、变量、超时长度如下:

- **>= 22.19.0**（`package.json` engines）。字段 `type: module` + `main: ./dist/index.js`, 强制 ESM。
- 默认无 DEEPSEEK_API_KEY / OPENAI_API_KEY 测试也能全部跑通: faux provider 做了可注入的测试桩。

## 7.5 Reading list

- `src/testing/storage-conformance.ts` — 自定义存储后端的全套验收标准。
- `src/testing/env-conformance.ts` — 自定义 `ExecutionEnv` 的验收标准。
- `test/harness-tasks.test.ts` — 任务模型完整测试。
