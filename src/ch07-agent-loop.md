# 第 7 章 agent-loop 精读：回合、步骤、收件箱

> **本章目标**
> 1. 逐行读懂 dsh 的默认驱动 `ReactLoopAgent`（`packages/core/agent-loop/src/agent.ts`）；
> 2. 深入状态机（phase）、收件箱（Inbox）、回合/步骤两层循环；
> 3. 看懂 `preStep` / `buildRequest` 的瀑布与请求冻结；
> 4. 与 agent-book 第 7 章对照，补上"工程化"更细的机制。

> **阅读前提**：请先读本书第 4-6 章和 agent-book 第 7 章（dsh 循环入门版）。
> 本章是进阶版：不再停留在"它做了什么"，而是拆到"状态机怎么转、消息怎么投"。

## 7.1 文件地图

`packages/core/agent-loop/src/agent.ts`（约 690 行）的核心成员：

| 成员 | 行号 | 作用 |
|------|------|------|
| `Phase` 类型 | 42 | 状态机：idle / maintenance / running |
| `ReactLoopAgent` 类 | 98 | 默认驱动 |
| `kick()` | 252 | 驱动器入口：`while (await this.turn()) {}` |
| `preStep()` | 267 | 打开步骤：领消息、组提示、走 `agent/pre-step` 瀑布 |
| `turn()` | 296 | 一个回合（回合级循环） |
| `step()` | 398 | 一个步骤：请求模型 + 处理输出 + 执行工具 |
| `prepareRequest()` | 547 | 解析请求配置并绑定适配器 |
| `buildRequest()` | 599 | 构造并冻结请求 |

配套文件：
- `tool-calls.ts`：工具调度器（并行/排他、超时、结果保序）；
- `runtime-context.ts`：运行时上下文投影；
- `inbox.ts`：收件箱——消息的领取、投递与唤醒。

## 7.2 状态机：`Phase`（42-49 行）

```ts
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | {
    kind: 'maintenance'
    abort: AbortController
    lastTurn: number
    wakeRequested: boolean
  }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

三个状态各回答一组问题：

| 状态 | 能不能收消息？ | 在干什么 | 取消会怎样 |
|------|--------------|---------|-----------|
| `idle` | 能（唤醒开新回合） | 无 | — |
| `maintenance` | 能（唤醒被锁存） | 压缩等维护任务 | 维护任务取消 |
| `running` | 能（按目标投递） | 正在跑回合/步骤 | 回合以 `aborted` 结束 |

`setPhase()`（145 行）统一管理迁移，并在状态改变时发 `agent/status` 事件。
**为什么要显式状态机？** 因为"Agent 在不在跑"不是布尔值——它可能在维护、
可能被取消后还有排队消息要唤醒。显式状态让"唤醒/取消/维护"都能精确表达。

## 7.3 驱动器与唤醒：`kick()` / `wakeDriver()`

```ts
private async kick(): Promise<void> {
  try {
    while (await this.turn()) {}       // 回合级循环：turn() 返回 true 就继续
  } catch (_error) {
    // 失败在驱动边界被收容（已在 agent/error 报告过）
  } finally {
    if (this.phase.kind === 'running') {
      const { turn, wakeRequested } = this.phase
      this.setPhase({ kind: 'idle', lastTurn: turn })
      if (wakeRequested && this.inbox.hasPending) this.wakeDriver()  // 有排队消息→再开
    }
  }
}
```

**唤醒（wake）机制**：`send(message, target, wakeup)` 在投递消息时，
如果当前不在 idle，就把 `wakeRequested` 锁存；驱动收敛到 idle 后，
看到锁存且有排队消息，就**再启动一个驱动器**。这实现了"跑完当前活，
自动接着处理新到的消息"。

注意一个细节：`kick()` 的 `catch` 故意吞掉错误——错误已经通过 `agent/error`
事件上报，驱动边界不再重复抛。这是"错误在活边界报告、在驱动边界收容"
的职责划分。

## 7.4 收件箱：消息的三种投递目标

dsh 用 `Inbox`（`packages/core/agent-loop/src/inbox.ts`）而不是简单的 `push`。
关键 API 是 `send(message, target, wakeup)`，`target` 决定消息何时被处理：

| 目标 | 含义 | 典型用途 |
|------|------|---------|
| `next-turn` | 下一个回合处理 | 普通用户消息、followup |
| `next-step` | 当前回合的下一步处理 | steer（转向）、工具结果上下文 |
| （splice 定位） | 更细粒度插入位置 | 在指定位置插入 |

`steer` 与 `followup` 的差别很有讲究：
- **steer → next-step**：立刻影响当前回合（用户中途"改成读 XML"）；
- **followup → next-turn**：下一回合再处理（不打断当前步骤）。

`turn()` 首步用 `target = 'next-turn'` 领消息，之后用 `'next-step'`。
工具结果通过回调放进 next-step 位置，让"工具结果 → 下一步"自然衔接。

## 7.5 `turn()`：回合级循环（296-396 行）

### 回合开头（296-311 行）

```ts
const { signal } = phase.abort
signal.throwIfAborted()                    // 302：每步先查取消
const turn = phase.turn + 1
this.session.append('turn/start', { turn })  // 305：日志：回合开始
let turnEnds: TurnEndReason | null = null
let target: InboxTarget = 'next-turn'
```

**第一步永远先 `throwIfAborted()`**——dsh 的取消检查遍布循环的每一个关键点，
保证"随时可取消、取消即停"。

### 回合主循环（313-364 行）

```ts
while (true) {
  signal.throwIfAborted()
  const step = phase.step + 1
  const decision = await this.preStep(target, { turn, step })  // 316：打开这一步
  if (decision.kind === 'reject') {          // 插件拒绝
    turnEnds = { kind: 'blocked' }
    return false
  }
  if (turnEnds && decision.messages.length === 0) break   // 321：已结束且无新消息
  if (phase.step === 0 && decision.messages.length === 0) {  // 324：首步就没消息
    turnEnds = { kind: 'completed' }
    return false                             // 不花模型调用，但保留回合边界
  }
  this.session.append('step/start', { turn, step })
  phase.step = step
  const toolRecovery = new ToolCallRecovery()               // 331：工具结果恢复器
  const stopRecovery = this.ctx.on('session/event', (session, event) => {
    if (session === this.session) toolRecovery.observe(event)
  })
  try {
    const stepEnd = await this.step(decision)
    if (turnEnds === null || turnEnds.kind !== 'max-tokens') turnEnds = stepEnd  // 341
  } catch (error: unknown) {
    // 步骤中途抛错：把已产生但未落盘的工具结果补记进日志，再原样抛
    for (const event of toolRecovery.results()) {
      this.session.append('tool/result', event.data, ...)
    }
    throw error
  } finally {
    stopRecovery()
    this.session.append('step/end', { turn, step })
  }
  signal.throwIfAborted()
  if (turnEnds && this.inbox.nextStep.length === 0) {
    await this.dispatch.serial('agent/turn-stopping', { turn, signal })  // 最后机会
    signal.throwIfAborted()
  }
  if (turnEnds && this.inbox.nextStep.length === 0) break
  target = 'next-step'                       // 还有工具结果/转向消息→下一步
}
```

要点：

1. **首步空消息也拥有回合边界，但不花模型调用**（324-327 行）：
   唤醒消息被移除/改写为空时，回合仍以 `completed` 关闭，日志记录"这次尝试"。
2. **`max-tokens` 是粘性的**（341 行）：某一步超长截断后，后续正常完成的步骤
   **不能把回合结果降级**——调用方始终知道"曾经被截断过"。
3. **`agent/turn-stopping` 是 serial**：回合要结束时，依次给每个插件
   "最后说句话"的机会（比如追加一条总结）。
4. **收件箱驱动下一步**：只要 next-step 里还有消息（工具结果、转向），
   回合就继续；否则 `break`。
5. **工具结果恢复（`ToolCallRecovery`，331-334 / 344-349 行）**：步骤内部会先
   产生工具结果、再落盘；若中途抛错，`catch` 块用恢复器把"已产生但还没写进日志"
   的结果补记进去，再原样抛出。**即使崩溃，工具副作用也有完整记录**。

### 回合收尾（366-396 行）

```ts
} catch (error: unknown) {
  const cause = abortedCancelCause(signal)     // 368：读取消原因
  if (cause !== undefined) {
    turnEnds = { kind: 'aborted', reason: cause }
    throw error
  }
  turnEnds = {                                     // 375：结构化错误
    kind: 'error',
    error: error instanceof LlmError
      ? error.failure
      : { message: errorChain(error), code: 'UNKNOWN' },  // 379
  }
  this.throwError(error)
} finally {
  this.session.append('turn/end', { turn, reason: turnEnds! })  // 385：无论如何都记
}
if (!this.inbox.hasPending) return false      // 390
phase.abort = new AbortController()           // 391
phase.wakeRequested = false                   // 393
phase.step = 0                                // 394
return true                                   // 395：还有下一回合
```

**取消原因**：`abortedCancelCause()` 只在信号确实被中止时返回原因，并从
`user / parent / disposed / hook` 四种里挑出**可序列化**的信息（`hook` 带
`reason`）——因为原始 `signal.reason` 可能被 fetch 塞进 `stack`，直接落盘会
失败。

**结构化错误**：`LlmError` 保留结构化 `failure`，其它错误压平成
`{ message, code: 'UNKNOWN' }`——保证 `turn/end` 永远有一个可序列化的
结束原因。`finally` 保证**即使异常，回合日志也完整**。

## 7.6 `step()`：步骤级循环（398-544 行）

新版把"解析请求配置"和"构造请求"拆成两个方法：`prepareRequest()`（547 行）
负责解析配置并绑定适配器，`buildRequest()`（599 行）负责构造并冻结请求。

```ts
private async step(decision): Promise<StepEndReason | null> {
  const { assembly } = decision
  const renderedPrompt = renderPrompt(assembly)        // 405：渲染系统提示
  let firstAttempt = true
  while (true) {
    const { config, preparedCall } = await this.prepareRequest(turn, step, signal)  // 408
    // ...写 system/message 日志；首次尝试写 user/message 日志...
    const request = this.buildRequest(config, preparedCall, assembly.tools,
      { turn, step }, startsRequestSeries, signal)     // 425：构造并冻结请求
    const live = new AssistantStreamAttempt(...)       // 426：本次"尝试"的追踪器
    let started = false
    try {
      const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
      signal.throwIfAborted(); live.start(); started = true
      for await (const chunk of stream) {              // 440：消费流式块
        signal.throwIfAborted(); live.push(chunk)
      }
    } catch (error: unknown) {
      if (!started) throw error
      if (signal.aborted) {                            // 被取消→落"中断消息"
        live.settle('assistant/message', () => this.session.append('assistant/message', {
          turn, step, message: createAssistantMessage({ content, source }),
          interrupted: true, ... }))
      } else {                                         // 流中断→落"尝试"
        live.settle('assistant/attempt', () =>
          this.session.append('assistant/attempt', { turn, step, stream: live.stream }).seq)
      }
      throw error
    }
    const finish = live.finish                         // 488：结束原因
    if (finish.kind === 'error' || finish.kind === 'aborted') {
      const action = await this.dispatch.waterfall(    // 494：请求错误瀑布
        'agent/request-error', { turn, step, provider: request.provider,
          failure: finish.failure, retryPolicy: preparedCall?.retryPolicy, signal },
        () => Promise.resolve<RequestErrorAction>(undefined),
      )
      if (action?.kind !== 'retry') throw new LlmError(...)  // 506-507
      continue                                         // 509：插件说重试 → 再来一轮
    }
    const message = createAssistantMessage({ ... })
    live.settle('assistant/message', () => this.session.append('assistant/message', {
      turn, step, message, ...live.usage, stream: live.stream }, { surfaceOp: 'append' }).seq)
    if (finish.kind === 'max-tokens') return { kind: 'max-tokens' }
    const toolCalls = message.content.filter(block => block.type === 'tool-call')
    if (toolCalls.length === 0) return { kind: 'completed' }
    const { concluded } = await executeToolCalls(      // 534：执行工具
      this.loopCtx, turn, step, toolCalls, signal,
      context => this.inbox.splice('next-step', this.inbox.nextStep.length, 0, [context]),
    )
    return concluded ? { kind: 'completed' } : null    // 538：null → while 继续
  }
}
```

**五个值得细品的点**：

1. **尝试（attempt）作为一等公民**：一次请求可能失败、重试、被取消。
   `AssistantStreamAttempt`（`live`）统一追踪本次尝试：对外发
   `agent/assistant-stream` 帧给 UI，结束时用 `settle` **结算**成一条日志事件。
   被取消时落一条 `assistant/message`（`interrupted: true`），把半截回复也记进
   日志；流中断则落 `assistant/attempt`，记录"这次尝试发生过"。**先落盘再消费**，
   因为"模型看到的一切都必须能从日志重建"。
2. **重试是插件的决定，不是循环的**：`agent/request-error` 瀑布让外部
   插件决定"重试还是放弃"。对比 codex 把重试写死在采样循环里（agent-book
   第 8 章）——这是"插件化"vs"内建"两种重试哲学的正面教材。
3. **工具结果进收件箱**：`executeToolCalls` 的结果上下文通过回调放进
   next-step 位置，让 `turn()` 的下一个步骤去领。
4. **`concluded` 语义**：某批工具结果"直接终结回合"时返回 completed，
   否则返回 `null` 让 while 继续（模型会看到工具结果后再被请求）。
5. **请求配置的解析（`prepareRequest`，新版独立出来）**：先读会话里已记录的
   请求头作为种子，恢复"该模型自己的"推理强度；再走 `agent/request` 瀑布允许
   插件改写；最后 `llm.prepareCall` 绑定适配器。**中间任何一步都要重新检查
   取消信号**。

## 7.7 `preStep()`：打开步骤的瀑布（267-286 行）

```ts
const claimed = this.inbox.claim(target, position.turn)      // 271：领消息
const assembly = await this.loopCtx.systemPrompt.assemble(assembleContextFor(this, signal))  // 272：组提示
const sections = renderContextSections(assembly)
const context = this.runtimeContext.project(joinContextSections(sections), sections)  // 275
const decision = await this.dispatch.waterfall(              // 276
  'agent/pre-step', { messages: claimed, ...position, signal },
  (): Promise<PreStepDecision> => Promise.resolve({
    kind: 'enter',
    messages: context === undefined ? claimed : [...claimed, context],
  }),
)
return decision.kind === 'reject' ? decision : { ...decision, assembly }
```

- **`claim` 是"领取"**：消息被标记为已被本回合处理，避免重复；
- **默认 enter**：消息 = 用户消息 + 运行时上下文投影（当前时间、环境等）；
- **waterfall 开放给插件**：改写 messages（注入上下文）或 reject（拒绝步骤）。

## 7.8 `buildRequest()`：请求的构造与冻结（599-687 行）

要点（详情见第 5 章"可重建请求"）：

1. 从 `session.requestHeader()` 派生初始配置（保持前缀稳定）；
2. 走 `agent/request` 瀑布让插件改写请求配置；
3. `prepareCall` 绑定"确切模型的适配器默认值"（如该模型默认 maxTokens）；
4. 请求与上下文变化时追加 `request/header` / `request/context` 日志事件；
   查询工具清单变化时追加 `developer/message`（`tool-addition` / `tool-removal`）
   ——**模型对工具的可见性变化本身也被记录**；
5. **请求 `deepFreeze` + `markAgentLoopRequest`**（677-686 行）：纯由日志派生、
   不可变、可逐字节重建。
## 7.9 本章小结

- 状态机 phase：idle / maintenance / running，`setPhase` 统一迁移并广播状态；
- 收件箱三目标：next-turn / next-step / splice，steer vs followup 语义分明；
- turn() 回合循环：preStep → 日志 → step → turn-stopping（serial）→ 判结束；
- step() 步骤循环：解析配置 → 构造请求 → 流式消费并结算尝试 → 错误可插件化重试 → 工具执行；
- max-tokens 粘性、ToolCallRecovery 补记、结构化错误、finally 记日志，都是"工程保险"。

## 动手练习

1. 对照源码，画出 `turn()` 里所有"return false / break / continue"的出口，
   标出各自的 `turnEnds` 值。
2. 注册一个 `agent/request-error` 瀑布监听器，模拟失败时返回 `{ kind: 'retry' }`，
   观察重试是否生效（结合 dsh 测试框架写个假模型）。
3. 思考题：为什么 `step()` 的 while 里用 `continue` 重试，而 `turn()` 的 while
   用 `break` 结束？这两种"循环控制"各自对应什么语义？

---

**下一章**：[第 8 章 LLM 能力层：适配器接缝与流式装配](./ch08-llm.md)
