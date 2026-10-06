# 附录 B 事件速查表

> 按事件域分类。模式：emit / waterfall / parallel / serial。完整权威列表见
> `docs/event-producer-consumer.md` 与各子系统页。

## Session 事件（持久，进日志）

| 事件 | 模式 | 含义 |
|------|------|------|
| `session/created` / `session/disposed` | emit | 会话生命周期 |
| `turn/start` / `turn/end` | emit | 回合边界（end 带 TurnEndReason） |
| `step/start` / `step/end` | emit | 步骤边界 |
| `user/message` | emit | 用户消息/注入上下文（surface: append） |
| `assistant/message` | emit | 装配完成的 assistant 消息（内嵌精确带时间流；surface: append） |
| `assistant/attempt` | emit | 未产出消息的尝试结算（失败/重试/取消/流中断） |
| `tool/result` | emit | 工具结果（surface: append） |
| `request/header` | emit | 请求非历史状态（EpochHeader） |
| `request/context` | emit | 路由容量（provider/model/contextWindow） |
| `agent/inbox/spliced` | emit | 收件箱持久变更（规范化 splice：start/removedCount/inserted） |
| `session/end-seed` | emit | 种子边界 |
| `session/flush` | parallel | 持久化检查点（回合结束排空写盘） |

## Agent 事件（运行期，携带活 Agent）

| 事件 | 模式 | 含义 |
|------|------|------|
| `agent/created` / `agent/disposed` | emit | 生命周期 |
| `agent/status` | emit | 状态变化（idle/running） |
| `agent/error` | emit | 失败在活边界上报 |
| `agent/assistant-stream` | emit | 运行期流式帧（attemptId + revision，不进日志） |
| `agent/inbox/inserted` / `discarded` / `claimed` | emit | 收件箱运行期通知（对应持久事件见 Session 表） |
| `agent/pre-step` | **waterfall** | 决定模型看到什么（改写/拒绝） |
| `agent/request` | **waterfall** | 决定请求配置 |
| `agent/request-error` | **waterfall** | 请求失败处置（可重试） |
| `agent/turn-stopping` | **serial** | 回合停止前依次给机会 |

## 能力事件（给接缝挂策略/适配器）

| 事件 | 模式 | 含义 |
|------|------|------|
| `llm/stream` | **waterfall** | 流式模型调用（重试/路由/重放） |
| `tools/pre-execute` | **waterfall** | 工具执行前（校验/策略） |
| `tools/post-execute` | **waterfall** | 工具执行后（加工/记录） |
| `tools/*`（其它） | 视声明 | 工具域策略 |
| `fs/*` | 视声明 | 文件系统策略 |
| `telemetry/*` | 视声明 | 遥测 |
| `system-prompt/assemble` | **waterfall** | 系统提示组装（scope 过滤） |
| `system-prompt/change` | emit | 提示分片变更通知 |
| `skills/change` | emit | 技能目录失效通知 |
| `approval/request` | **waterfall** | 审批作答 |
| `approval/asked` / `approval/decided` | emit | 审批审计对 |

## 选择口诀

- **改决策** → waterfall / serial（`next()` 委托或短路）；
- **只通知** → emit；
- **并发收尾** → parallel；
- **持久事实** → Session 事件；**运行期拦截** → Agent / 能力事件。
