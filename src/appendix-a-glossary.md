# 附录 A 术语表

> dsh 术语速查。括号里是详述章节。

## A

**Agent（智能体）**：一个会话的驱动器（`Agent` 接口），由 `agent-loop` 实现。（第 6 章）

**Agent Loop（agent-loop）**：默认驱动 `ReactLoopAgent`，回合/步骤循环。（第 7 章）

**Agent Note**：dsh 的决策记录（`.agents/notes/`），记录"问题→决策→备选→后果"。（第 19 章）

**Agent scope**：把注册限定到单个 agent 的作用域（`agent.ctx`）。（第 13 章）

**append-only log（追加式日志）**：只追加、不改写的日志；会话的真相源。（第 4 章）

**Approval（审批）**：回答"这个动作能不能做"，fail-closed。（第 15 章）

## B

**Branded ID**：跨边界不透明 id（`Branded<B>`），如 `CallId`、`ApprovalRequestId`。（第 15 章）

**Bundle（分发包）**：可分发的能力包（base / web-app / headless）。（第 1、17 章）

## C

**Capability seam（能力接缝）**：Service Definition / Provider / Consumer 三角色。（第 10 章）

**Compaction（压缩）**：用摘要替换"表面"而不改日志。（第 14 章）

**Context（上下文）**：`ctx`，服务的仓库；插件按键找服务。（第 2 章）

**Cordis**：dsh 底下的插件框架（vendored）。（第 2 章）

**Consumer（消费者）**：接缝中"使用能力"的一方，通常是模型可见工具。（第 10 章）

**cordis.yml**：组合插件树的编排文件。（第 17 章）

## D

**defineTool**：定义工具的最小形态（schema + execute + output）。（第 18 章）

**deriveMessages()**：从会话日志派生模型可见消息。（第 4 章）

**disposer（卸载器）**：`register()` 返回的清理函数；注册即副作用。（第 2 章）

## E

**EpochHeader**：记录请求的非历史状态（配置/系统提示/工具 schema）。（第 5 章）

**Event sourcing（事件溯源）**：日志即状态，其它一切由日志派生。（第 4 章）

**execute()**：工具的执行函数，遵守严格契约。（第 18 章）

**exclusive（排他）**：工具执行模式，形成屏障。（第 9 章）

## F

**fail-closed（默认拒绝）**：除非显式放行否则不放行。（第 15 章）

**fail loud（大声失败）**：错误配置/不支持的能力要大声报错，不静默降级。（第 12、17 章）

## I

**Inbox（收件箱）**：消息投递管理（next-turn / next-step）。（第 7 章）

**inject**：插件声明服务依赖的方式。（第 2 章）

**Initiating Agent（发起者）**：当前工作的归属 agent。（第 6 章）

**invariant（不变量）**：包自有运行期检查（`ctx.invariants`）。（第 16 章）

**isolate realm**：preset 里隔离服务实例的作用域。（第 17 章）

## L

**llm-replay**：可回放的 LLM 适配器，支撑快照测试。（第 8 章）

**Model-visible ⟺ logged**：模型可见的必可记录。（第 5 章）

## P

**parallel（并行）**：工具执行模式，有界并发池。（第 9 章）

**Phase**：Agent 状态机（idle / maintenance / running）。（第 7 章）

**Preset（预设）**：按会话组装能力集（agent-presets）或权限组合（permission-presets）。（第 15、17 章）

**Profile（配置文件）**：一个产品形态的 bundle 组合。（第 1 章）

**Provider（服务提供者）**：实现接缝接口的一方。（第 10 章）

## R

**ReactLoopAgent**：dsh 默认的回合/步骤驱动实现。（第 7 章）

**Reconstructable request（可重建请求）**：任何模型请求可从日志逐字节重建。（第 5 章）

**Registration is an effect（注册即副作用）**：通过 ctx.effect/on 注册，register 返回卸载器。（第 2 章）

**resolve()**：把 request 显式补全成 spec（显式 > 隐式）。（第 11 章）

## S

**Sandbox（沙箱）**：约束文件副作用（read-only / workspace-write / danger-full-access）。（第 15 章）

**Scope（作用域）**：把注册限定到某个 agent 的原语。（第 13 章）

**Seam（接缝）**：见 Capability seam。（第 10 章）

**Service Definition（服务定义）**：接缝的接口声明。（第 10 章）

**Session（会话）**：追加式事件日志 + 派生消息历史。（第 4 章）

**SessionEventMap**：会话事件词汇，合并可扩展。（第 4 章）

**spill（溢出）**：超大工具输出溢出到存储，会话只留摘要。（第 9、14 章）

**Step（步骤）**：一次模型请求 + 它调用的工具。（第 7 章）

**Surface（表面层）**：从日志折叠出模型可见消息的视图。（第 4 章）

**SurfaceOp**：事件如何进入表面（append / replace）。（第 4 章）

## T

**Tool（工具）**：模型可调用的能力 = schema + execute + output。（第 9、18 章）

**Turn（回合）**：零或多个步骤；从开口到无欠账关闭。（第 7 章）

**TurnEndReason**：回合为什么结束（completed / max-tokens / error / aborted / blocked）。（第 7 章）

## W

**Waterfall（瀑布）**：可改写/可短路的事件分发；`next()` 委托。（第 3 章）
