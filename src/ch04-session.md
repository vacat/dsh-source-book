# 第 4 章 会话与事件溯源：日志即真相

> **本章目标**
> 1. 理解 dsh 会话的核心：追加式事件日志（event sourcing），日志即真相；
> 2. 掌握 `SessionEventMap` 事件词汇与 `SessionEvent` 结构；
> 3. 理解"表面层（surface）"如何从日志折叠出模型可见消息；
> 4. 理解格式版本机制（`SESSION_FORMAT_VERSION` 与 `ignorable` 标记）。

## 4.1 生活化开场：流水账 vs 结论

第 11 章（agent-book）我们已经对比过"状态快照"与"事件日志"。
dsh 选择的是**事件日志，而且是追加式、只增不改的**。它的哲学浓缩在
Agent Note `2026-06-11-event-sourced-sessions` 里：

> A `Session` is an append-only log of typed `SessionEvent`s — the single
> source of truth. The LLM message history is *derived* from the log
> (`deriveMessages()`).

关键洞察：**日志不是"状态的记录"，日志就是状态。** 任何别的表示（内存消息、
持久化文件、UI 视图）都是从日志**派生**出来的视图。既然状态=日志，
"状态与日志不一致"这类 bug 在结构上就不存在。

## 4.2 会话模块：`core/session`

`packages/core/session/` 的模块头注释一句话讲清了它的三个组成：

> **Event-sourced session service: append-only session log, in-memory store,
> and the derived LLM message history.**

| 组成 | 是什么 |
|------|--------|
| 追加式日志 | `session.append(type, data, opts)` 追加 `SessionEvent` |
| 内存 store | 会话的运行时容器（`ctx.sessions`） |
| 派生消息历史 | `deriveMessages()` 从日志投影出模型可见的消息 |

**注意注释里的一句**：持久化是插件关注点（Persistence is a plugin concern）。
`core/session` 只提供内存日志；落盘由 `session-persistence-jsonl` 插件订阅
`session/event` 并异步写库，
在回合结束时等待 `session/flush` 检查点刷盘（见第 14 章）。

## 4.3 事件词汇：`SessionEventMap`

会话里能记什么，由 **`SessionEventMap`** 这个类型定义。它是一张
"合并可扩展（merge-extensible）"的映射：核心包定义了一批事件，
插件可以**声明合并**往里加自己的事件类型。看 `docs/subsystems/session.md`
列的常见类型：

| 事件 | 含义 |
|------|------|
| `session/created` / `session/disposed` | 会话生命周期 |
| `turn/start` / `turn/end` | 回合边界（`turn/end` 带 `TurnEndReason`） |
| `step/start` / `step/end` | 步骤边界 |
| `user/message` | 用户消息 / 注入上下文（surface 事件） |
| `assistant/message` | 装配完成的 assistant 消息（surface 事件，**内嵌精确带时间流**） |
| `assistant/attempt` | 未产出消息的尝试结算（失败/重试/取消/流中断） |
| `tool/result` | 工具结果（surface 事件） |
| `request/header` | 请求的非历史状态（调用配置、系统提示、工具 schema） |
| `request/context` | 请求的路由容量（provider/model/contextWindow） |

**为什么把流也记进日志？** 两个原因：
1. **token 级回放保真**——`assistant/message` 内嵌该次请求的精确带时间流
   （`AssistantStreamAttempt` 逐块累积），UI 逐字重放、调试都靠它；
2. **失败的尝试也有记录**——没有产出消息的尝试（失败/重试/取消/流中断）
   落成 `assistant/attempt`，日志里"这次尝试发生过"可查。

> **AGENTS.md 关键规则**：`SessionEventMap` 的成员**默认"读了就必须知道"**
> （required-on-read）。不认识它的构建会拒绝读取日志，除非事件携带
> `ignorable: true`。这条规则保证了"未知事件不会悄悄丢内容"（见 4.6 节）。

## 4.4 表面层（surface）：日志 → 模型可见消息

日志里不是所有事件都该给模型看（比如 `turn/start` 是管理事件）。dsh 用
**表面层（surface）** 从日志中挑选、折叠出"模型真正看到的消息"。
`packages/core/session/src/surface.ts`：

```ts
/** 会产出消息的事件联合在运行期的对应物（surface 子集） */
const SURFACE_EVENT_TYPES = new Set<string>([
  'system/message',
  'developer/message',
  'user/message',
  'assistant/message',
  'tool/result',
])

/** 一个事件类型能否进入模型可见表面 */
export function isSurfaceEligibleType(type: string): boolean {
  return SURFACE_EVENT_TYPES.has(type)
}
```

**模型可见内容只流经这五种 surface 事件**（加上 `request/header` /
`request/context` 折叠出的非历史状态）。这条约束非常关键：
它让"安全分析"变得可计算——第 5 章的"模型可见 ⟺ 可记录"之所以能成为
不变量，就是因为模型能看到的入口是**封闭的**。

`SurfaceOp` 描述事件以什么方式进入表面：

| `SurfaceOp` | 含义 |
|-------------|------|
| `'append'` | 追加到消息尾部（普通路径） |
| `{ op: 'replace'; start; end }` | 用摘要替换一段旧消息（压缩时用） |

压缩（第 14 章）就是靠 `replace` 把长历史折叠成摘要，而日志仍保留原始事件
——**压缩改的是"表面视图"，不改"日志真相"**。这是"日志即真相"在压缩上的体现。

## 4.5 派生：`deriveMessages()`

`Session.deriveMessages()` 把表面事件折叠成发给模型的消息数组。
Agent Note `2026-07-05-reconstructable-requests` 说明它的实现要点：

- **每个表面条目只投影一次**：通过公开的纯函数 `deriveEventMessage(event)`，
  首次看到时投影并缓存；表面被改写（压缩 `replace`）时重建；
- **每次调用返回新数组**，但共享深冻结（deep-frozen）的消息对象——
  通过投影修改历史**在类型上不可表示**（会抛错）；
- **外部重建者**用同一个公开函数折叠日志前缀，所以"运行时投影"和
  "离线重建"两条路径不会得出不同结果。

**这条"单一投影函数、两条路径一致"的设计**，是 dsh 可复现性承诺的技术保障
（第 5 章深入）。

## 4.6 格式版本：`SESSION_FORMAT_VERSION` 与 `ignorable`

日志会跨版本演进。dsh 的版本机制（Agent Note `2026-08-10-session-log-version-mechanism`）
有几个值得学的决策：

1. **单个单调整数，不分主/次**（`SESSION_FORMAT_VERSION`，当前为 `4`）：
   "能不能自动升级"是**每一步 upgrader 是否存在**的属性，不该用数字形状承诺；
2. **写者决定何时 bump，不是读者**：当"旧运行时无法正确处理新日志"时才必须
   bump；"能解析"不算数——"静默跳过会影响重建的内容"是错误读法；
3. **方向化读取**：版本相等正常读；比读者新则拒绝并指明方向（"由更新的
   harness 写入——请升级"），同时提供原始日志路径让用户仍能看到文本；
   比读者旧则通过 `n→n+1` upgrader 链在内存中转换；
4. **`ignorable` 标记覆盖词汇增长**：普通事件新增**永不 bump 版本**——
   读者遇到不认识的事件类型时，除非事件带有 `ignorable: true`，否则拒绝读取。
   **默认是"必读"**：忘记标记的代价是"过度拒绝一个可恢复会话"（小麻烦）；
   而默认可忽略的代价是"静默恢复一个被掏空的会话"（安全失败）。dsh 选择
   前者的代价，换取后者绝不发生。

> **已发布的格式世代不可改写**：一旦某个格式随产品发布（released），它就冻结成
> "不可变的旧代（immutable prior generation）"，只能**追加**相邻的
> `n→n+1` 转换包（仓库里的 `session-format-v0-to-v1` … `v3-to-v4` 就是这些
> 转换），**绝不移动、覆盖或删除**已提交的世代。当前工作区写入器是 `4`、
> 最新已定版（finalized）也是 `4`，而最新**已发布**（released）记录为 `3`
> （依据 `docs/session-format-status.md`）。

> **一句话**：版本机制的目标不是"尽量兼容"，而是"**要么正确读，要么大声拒绝**"。

## 4.7 本章小结

- 会话 = 追加式事件日志，日志即真相；消息历史从日志派生；
- `SessionEventMap` 是合并可扩展的事件词汇；`assistant/message` 内嵌精确带时间流保证 token 级回放，`assistant/attempt` 记录未产消息的尝试；
- 表面层（surface）：模型可见内容只流经 system/message、developer/message、
  user/message、assistant/message、tool/result 五种 surface 事件 + request 折叠；
- `deriveMessages()` 单一投影函数，运行时与离线重建路径一致；
- 版本机制：单整数、写者决定 bump、"要么正确读要么大声拒绝"、`ignorable` 标记。

## 动手练习

1. 读 `packages/core/session/src/types.ts`，数一数 `SessionEventMap` 里
   有多少个事件类型，标出哪些带 `surfaceOp`。
2. 读 `packages/core/session/src/surface.ts` 的 `foldSurface`，理解
   `replace` 折叠是怎么工作的。
3. 思考题：为什么 `ignorable` 的默认是"必读"而不是"可忽略"？
   （答案见附录 C，结合"静默恢复被掏空的会话"想。）

---

**下一章**：[第 5 章 模型可见 ⟺ 可记录：请求可重建](./ch05-reconstructable.md)
