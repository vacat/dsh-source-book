# 第 8 章 LLM 能力层：适配器接缝与流式装配

> **本章目标**
> 1. 理解 `ctx.llm`（`LlmRuntime`）的定位：模型适配器注册表 + 流式调用 API；
> 2. 掌握 `StreamChunk` 词汇与 `BlockAssembler` 的装配逻辑；
> 3. 理解适配器实现：llm-deepseek / llm-pi-ai，以及测试侧的 llm-replay；
> 4. 理解 retry 策略与 token 计量如何接入。

## 8.1 `ctx.llm`：一个"适配器注册表 + 流式调用"

`packages/llm/llm/src/index.ts` 模块头：

> **LLM service: adapter registry with a waterfall-interceptable streaming call
> API.** Exports the `LlmRuntime` default, the abstract `LlmAdapter` for
> provider backends, and `BlockAssembler` for chunk assembly.

三个关键角色：

| 角色 | 是什么 |
|------|--------|
| `LlmRuntime` | `ctx.llm` 默认服务：注册适配器 + `stream()` / `prepareCall()` |
| `LlmAdapter` | 抽象基类：provider 后端要实现它 |
| `BlockAssembler` | 把流式块装配成内容块 |

它声明了两个 `ctx` 扩展（`ctx.llm` 服务 + `llm/stream` 事件），并强调
`llm/stream` 是 **waterfall**——任何插件都能在真正调模型前插入逻辑
（重试、路由、缓存、重放）。

## 8.2 请求词汇：`GenerateOptions`

模型请求被统一成一个结构（`packages/llm/llm/src/types.ts`）。核心字段：

```ts
export interface GenerateOptions {
  provider: string                       // 注册的路由，选择适配器实例
  model: string
  reasoningEffort?: ReasoningEffortId
  messages: RequestMessage[]             // 顺序对话消息（loop 从日志派生）
  system?: string                        // 系统提示（适配器映射到 provider 的系统槽）
  tools?: ToolSchema[]                   // 工具 schema
  toolHistory?: ToolHistory              // 已记录的工具增删历史（路由投影用）
  temperature?: number
  maxTokens?: number
  stop?: string[]
  signal?: AbortSignal
  sessionId?: Branded<'SessionId'>       // 会话标识（回放时分离游标）
  purpose?: 'compaction' | 'session-title'  // 辅助模型调用的分类
}
```

**注意 `sessionId` 与 `purpose`**：前者让回放能分离游标；后者把"压缩调用 /
标题生成调用"和"普通对话调用"区分开——适配器可据此走不同的生成策略。
这就是第 5 章"请求可重建"里 `EpochHeader` 记录非历史状态的具体载体。

## 8.3 流式词汇：`StreamChunk`

模型输出被标准化成一连串 `StreamChunk`。**这是一个合并可扩展的联合**，
插件可以加自己的 chunk 类型。核心成员：

```ts
| { type: 'block-start'; index: number; blockType: ContentBlockType }  // 内容块开始
| { type: 'text-delta'; index: number; text: string }                 // 文本增量
| { type: 'reasoning-delta'; index: number; text: string }            // 思考增量
| { type: 'tool-call-delta'; index: number; id: ToolCallId; name?: string; argumentsDelta: string }
| { type: 'block-end'; index: number; block: ContentBlock }           // 内容块结束
| { type: 'usage'; usage: TokenUsage }
| { type: 'finish'; reason: ... }
```

**为什么工具参数也用"delta"**：模型的工具参数是流式生成的 JSON 文本，
`argumentsDelta` 逐段累积，直到 `block-end` 才得到完整参数。这正是
第 7 章 `BlockAssembler` 要做的事——把增量拼成完整块。

## 8.4 `BlockAssembler`：从增量到块

`packages/llm/llm/src/assembler.ts` 的 `BlockAssembler` 是"流→块"的装配器：

- `push(chunk)`：喂一个 `StreamChunk`，内部按 `index` 定位到对应内容块；
- `blocks()`：取装配好的完整内容块数组；
- `finish`：结束原因（completed / max-tokens / error / aborted）；
- `usage`：token 用量。

第 7 章 `step()` 里，装配被包在 `AssistantStreamAttempt`（`live`）里：
`live.push(chunk)` 先给块打上时间戳存进持久流累积器，再把同一份块喂给
`BlockAssembler` 装配内容块，最后对外发一帧 UI 事件；请求成功时
`live.settle('assistant/message', ...)` 把
`live.stream`（精确带时间流）和 `live.blocks()`（装配好的内容块）一起写进
同一条事件：

```ts
const live = new AssistantStreamAttempt(session.id, ++attempt, ...)
for await (const chunk of stream) {
  live.push(chunk)                    // 累积精确流 + 装配内容块 + 发 UI 帧
}
live.settle('assistant/message', () =>                           // 成功
  this.session.append('assistant/message',
    { turn, step, message, stream: live.stream }, { surfaceOp: 'append' }).seq)
live.settle('assistant/attempt', () =>                           // 失败/取消
  this.session.append('assistant/attempt', { turn, step, stream: live.stream }).seq)
```

**注意"流"与"块"在同一条事件里**：`assistant/message` 既带装配好的完整消息
（供模型和 surface 使用），又带精确的带时间流（供逐字回放）；没有产出消息的
尝试则单独落 `assistant/attempt`。

## 8.5 适配器实现

dsh 的 `packages/llm/` 下的上市适配器有两个（`llm-deepseek` / `llm-pi-ai`），
另有回放用的 `llm-replay` 落在测试支持组里——它们正好构成一条"演进"链：

### llm-deepseek：第一方适配器

`packages/llm/llm-deepseek/src/adapter.ts` 直接对接 DeepSeek API：
`serialize.ts`（把 `GenerateOptions` 序列化成 DeepSeek 请求体）、
`sse.ts`（SSE 流解析）、`translate.ts`（响应翻译回 `StreamChunk`）。

### llm-pi-ai：复用 pi 的 provider 生态

`packages/llm/llm-pi-ai/` 是一个有趣的适配器：**它复用 pi 项目的
`packages/ai` 多 provider 生态**（OpenAI/Anthropic/Google…几十家），
把 pi 的事件流翻译成 dsh 的 `StreamChunk`。文件里有 `provider.ts`（provider
路由）、`stream.ts`（流翻译）、`catalog.ts` / `discovery.ts`（模型发现）、
`replay.ts`（回放支持）。

> **设计意义**：dsh 不需要为每家模型重写适配器——它"借"pi 已维护的
> provider 生态，只写一层翻译。这是"用成熟依赖替代自造轮子"的实例
> （对应 AGENTS.md 的依赖优先策略）。

### llm-replay：可回放的适配器

`packages/test-support/llm-replay/` 用于**回放**：它短路 `llm/stream`，把
录好的会话 JSONL 里的模型块重建成"模型输出"，配合快照测试在无 key 时验证
行为。它是"请求可重建"在测试侧的延伸，所以归在 `test-support` 而不是
上市适配器组——第 16 章再展开。

### retry：`llm-retry` 包

`packages/llm/llm-retry/` 提供 `retry-policy.ts`（重试策略）。注意：**重试
不是写死在适配器里**，而是通过 `agent/request-error` 瀑布（第 7 章）由
插件决定——`retryPolicy` 作为瀑布输入暴露给插件。

## 8.6 token 计量：`token-meter`

`packages/llm/token-meter/` 提供 `ctx.tokenMeter`：**重放时的 token 计量**。
因为"请求可重建"，回放时可以精确计算每次请求消耗了多少 token
（对成本核算、预算控制有用）。

## 8.7 三库对照：dsh 的 LLM 层 vs pi / codex

| 维度 | pi | dsh | codex |
|------|-----|-----|-------|
| 统一接口 | `Models.streamSimple` | `ctx.llm.stream` | `ModelClient` |
| 适配器 | providers/*.ts（几十家） | llm-deepseek / llm-pi-ai | model-provider 等 |
| 流词汇 | AssistantMessageEvent | StreamChunk | Responses API item |
| 装配 | 事件里带 partial 快照 | BlockAssembler 显式装配 | 官方 item 流 |
| 重试 | 无内建 | agent/request-error 瀑布 | 内建状态机 |
| 可回放 | 无 | llm-replay 适配器 | 部分 |

**dsh 最独特的一点**：把"可回放"做成了适配器本身（llm-replay）——这源于
"请求可重建"的不变量（第 5 章）。

## 8.8 本章小结

- `ctx.llm` = 适配器注册表 + 流式调用 API + `llm/stream` 瀑布；
- `GenerateOptions` 统一请求，`StreamChunk` 统一流，`BlockAssembler` 装配；
- 适配器：deepseek（第一方）、pi-ai（复用生态），另有测试侧的 replay（回放）；
- 重试走 `agent/request-error` 瀑布（插件决定），token 计量走 `token-meter`；
- "可回放即适配器"是 dsh 独有的设计。

## 动手练习

1. 读 `packages/llm/llm/src/index.ts`，找 `LlmRuntime` 的 `stream()` 与
   `prepareCall()` 签名，理解两者的分工。
2. 对比 `llm-deepseek/src/serialize.ts` 与 `llm-pi-ai/src/stream.ts`，
   找出两处"把 dsh 请求翻译成 provider 请求"的差异。
3. 思考题：为什么 `BlockAssembler` 要按 `index` 定位内容块，而不是简单地
   顺序累积？提示：模型可能交错输出文本块与工具调用块。

---

**下一章**：[第 9 章 工具系统：注册表与执行流水线](./ch09-tools.md)
