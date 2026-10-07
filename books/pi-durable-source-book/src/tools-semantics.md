# 第 6 章 工具执行与重放

工具调用是 pi-durable 的第二张底牌: **每一次工具调用当作一个持久化任务, 而不是一个黑洞 async 函数**。本章看工具生命周期的七道工序: 提交意图、验证参数、执行、结果落盘、abort、重放与重试。

## 6.1 一轮工具调用的事件流

模型回复后, pi-durable 并不是“在内存中直接运行工具”。`harness/generation.ts` 的 `afterTools()` 显示, 工具结果生成后先分析是否终止运行:

```ts
// generation.ts:593-615 简化
await runtime.hooks.each("afterTools", (hook) => hook(assistant, results, runtime, context));
// Every call of the round, including those answered without a task, must ask to terminate.
const terminate = slots.every((slot) => slot.taskId !== undefined && controls.get(slot.taskId)?.terminate === true);
```

`terminate` 是工具请求 `control: { terminate: true }` + 每一轮**所有调用**都要求才生效——因为需要一个模型回合内的所有调用**认同**终止。

## 6.2 回合与任务:一次 tool call 的一生

样例, 来自 README (中文解释), 一个工具调用即一个任务:

```text
submit(input) → pi.user
  pi.generation → pi.system (only if the prompt or tools changed), pi.assistant (tool calls)
    pi.tool × n → pi.tool-result × n   (owned by the generation, which waits for them)
  pi.generation → pi.assistant (answer) → submission done
```

`pi.tool` 是内置任务的典型: 工具执行阶段自带 checkpoint (执行之前把**执行状态 + args + replay 信息**写进存储)。

**Harness 只有在 checkpoint 写完后才执行工具本体 `execute(args, api)`**:

```ts
// Authors may return a final result instead of concurrent module:
if (!tool?.execute) { /* no task for it */ }
```

如果进程崩溃发生在 execute 中间:

- `replay: "safe"` 的工具: 重启重入并重新 execute。
- `replay: "unsafe"` 的工具: 不予重放, 交回给模型一个"中断了"的失败结果。

这是 pi-durable 上线后的“失去这个任务是否要多花一次调用”的假设: **不用作 fallback, 无重试再提交**。

## 6.3 工具执行中的中止

pi-durable 通过 `Context` 传递 abort 信号, `chord/context` 的 `withAbortSignal` 负责把信号注入到任务。`ToolTask` 支持 abort: 回合结束时调用 `tool.abort()` 区分结果的终止与再执行。

## 6.4 输出流与断点续传

pi-durable 的 tool 有"流式进度写入" — `api.output(text)` 直接 append 到任务内部 buffer, `output(chunk, skipped)` 可声明"已跳过"。README 原话: "`api.output()` streams running output, which becomes the result when `execute()` returns no `content`."

一个 tool 不返回 content 而是反复 `api.output()` 时, 该 output 就是最终 result。默认每 100ms commit 一次:

```ts
// src/harness/output.ts
export const PROGRESS_BYTES_PER_SECOND = ...
```

## 6.5 从示例代码看计时 Hook

`test/examples/17-coding-tools.ts` 演示了"计时 Hook":

```ts
// 17-coding-tools.ts:56-67 简化
const Timing = defineExtension({
  hooks: [
    hook(ToolTask, {
      beforeTool: (call) => { if (isBigCat(call)) start = Date.now(); },
      afterTool: (call) => { if (isBigCat(call)) console.log(`took ${Date.now() - start} ms`); },
    }),
  ],
});
```

这个例子同时展示了两件事: **Hook 也走 `ToolTask`**, 并生成 `pi.tool-result` per call。

## 6.6 Reading list

- `src/harness/tool.ts:27`—`ToolTask` 定义。
- `src/harness/tool.ts:62`—`call` 阶段的执行链: resolve tool → validate → beforeTool → execute → afterTool。
- `src/harness/tool.ts:184`—`settle()` 与 abort 状态。
- `src/harness/generation.ts:593`—`afterTools()`。
- `src/tools/bash.ts`—`bash` 工具的 execute 与输出流示例。
