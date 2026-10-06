# 附录 D 延伸资源

> 继续深入 dsh 的资源清单。以 commit `49120665f`（2026-09-26）为基准。

## 1. dsh 仓库自带文档（第一手）

- `docs/architecture.md` —— 架构总览（必读，改 packages 前先读）
- `docs/cordis-primer.md` —— Cordis 入门（本书第 2 章）
- `docs/capability-seams.md` —— 能力接缝服务图（代码生成，永不过时）
- `docs/subsystems/*.md` —— 每个子系统的服务/事件/API 参考
- `docs/event-producer-consumer.md` —— 事件地图
- `docs/config-catalog.md` —— 配置目录（代码生成）
- `docs/glossary.md` —— 官方术语表
- `docs/cookbook/` —— 扩展指南（adding-a-tool / adding-a-package / adding-an-llm-adapter）
- `docs/testing.md` —— 测试策略（第 16 章）
- `.agents/notes/README.md` —— Agent Notes 规范（第 19 章）
- `.agents/notes/implemented/architecture/` —— 架构决策记录（阅读金矿）

## 2. 按主题推荐的 Agent Notes

| 主题 | 推荐阅读 |
|------|---------|
| 会话 | `docs/subsystems/session.md`；Agent Note `2026-06-11-event-sourced-sessions`、`2026-06-18-session-surface` |
| 请求可重建 | `2026-07-05-reconstructable-requests` |
| 版本机制 | `2026-08-10-session-log-version-mechanism` |
| agent 生命周期 | `docs/subsystems/core.md`（Creation and ownership） |
| scope | `docs/subsystems/scope.md` |
| 能力接缝 | Agent Note `2026-06-13-capability-seams`、`2026-06-24-web-capability-seam`；`docs/subsystems/filesystem.md` |
| LLM 适配器 | `2026-06-13-twin-llm-adapters`、`2026-07-14-provider-routed-llm-adapters` |
| 工具 | `2026-07-07-tool-call-timeout-policy`、`2026-07-08-tool-output-spill-files`、`2026-07-02-tool-render-intent-union` |
| 持久化 | `docs/subsystems/persistence.md` |
| 压缩 | `docs/subsystems/compaction.md` |
| 子代理 | `docs/subsystems/subagent.md`；Agent Note `2026-07-28-continuable-subagent-conversations` |
| 测试 | `docs/testing.md`；Agent Note `2026-06-19-real-api-e2e-ci` |
| 流程 | `.agents/notes/README.md`；Agent Note `2026-07-26-frozen-agent-note-archive` |

> **关于归档**：`docs/subsystems/*.md` 是**代码生成的当前权威**。上表原先
> 推荐的若干 Agent Note（`2026-06-14-session-persistence`、
> `2026-06-17-filesystem-capability-seam`、`2026-06-18-agent-lifecycle-and-ownership-contracts`、
> `2026-06-19-acp-snapshot-tests`、`2026-06-21-subagent-capability-seam`、
> `2026-07-05-uniform-agent-note-format`、
> `2026-07-10-after-call-compaction-pressure-and-overflow-recovery`、
> `2026-07-12-agent-scope-runtime-design`、`2026-07-12-scoped-layers-store`、
> `2026-08-08-bounded-session-persistence-write-batching`、
> `2026-08-13-session-content-search-opt-in`）已于 2026-09-04 / 2026-09-30
> 归档到 `.agents/notes/archived/`：它们**永久冻结、不再作为当前行为依据**，
> 只在你需要历史决策时去读。

## 3. 配套书籍

- **agent-book**：《Agent 技术实战》——本书的前置入门
- **llm-inference-book**：《大模型推理技术详解》——模型推理侧源码

## 4. 实践建议

1. **先读再改**：改任何 `packages/` 前，先读 `docs/architecture.md` 与相关
   subsystem 页；
2. **用 Agent Notes 驱动阅读**：第 19.4 节的"决策驱动阅读"；
3. **跑门禁前只跑相关检查**：别全量跑（第 16 章）；
4. **非平凡改动写 Agent Note**：这是 dsh 的硬规则，也是你理解它的最好方式。

## 5. 注意

代码与文档持续演进。以你本地 `git -C <dsh> rev-parse HEAD` 的实际版本为准；
行号与文件路径可能漂移。
