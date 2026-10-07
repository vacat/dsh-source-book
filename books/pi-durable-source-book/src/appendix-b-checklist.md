# 附录 B 学习清单

把全书抽成"读完应该能回答的问题"。带上勾选框读, 是最不费力的自检。

## 总体（第 1、2 章）

- [ ] 说出 pi-durable 与 pi-ai、pi-agent-core 各自承担的层。
- [ ] 画出 pi-durable 内部的四层(Session / Store / Harness / Edge)与之间的调用方向。
- [ ] 说出 chord 在 pi-durable 中的角色。

## Session 层（第 3 章）

- [ ] 解释 mutation line 为什么是 pi-durable 排序的底层构造。
- [ ] 说出一座 `commit()` 从 `Tx` 到 publish 的典型生命周期。
- [ ] 列出 `#poison` 可被触发的至少 2 种场景。
- [ ] 解释为什么 **文档版本对 `storedVersion` / `valueVersion`** 必须同时存在。

## Store 层（第 4 章）

- [ ] 列出 Memory / SQLite / JSONL 三后端在**提交点**上的语义差。
- [ ] 解释 JSONL 里"marker"的作用。
- [ ] 假设你的后端崩溃时丢失了部分 WAL, 上述三种后端哪个更显式处理这个边界。

## Harness 层（第 5 章）

- [ ] 从 `root.submit()` 出发, 顺出一次"用户输入 → 模型回答落盘"的调用栈。
- [ ] 区分 `Submission`、`Conversation`、`Task` 三个对象在生命周期意义上的差别。
- [ ] 说出 `pi.tool` / `pi.generation` 两个内建任务必要的 phase。

## 工具层（第 6 章）

- [ ] 解释 `afterTools` 为什么必须**所有调用**都 `terminate: true` 才结束一轮。
- [ ] 区分 `replay: safe` 与 `replay: unsafe` 对崩溃恢复的影响。
- [ ] 用 `test/examples/17-coding-tools.ts` 说明"Hook 也在 tool 流程中"。

## 任务抽象 / Storage 契约（第 8 章）

- [ ] 写出 `TaskOptions` 的两个关键字段与语义。
- [ ] 解释为什么 `task` 记录在存储层是"整条 JSON"而不是 diff。
- [ ] 说出 `Page` / `Cursor` 双方的控制权归属。
- [ ] 区分 `ConversationRecord` 的 `parent` 和 `owner`。

## 扩展层（第 9 章）

- [ ] 解释 `wrapTool` 与"同名覆盖"的区别。
- [ ] 使用 `defineTool` 实现一个 `replay: safe` 工具, 并说明为什么它是安全的。
- [ ] 用 `defineDoc` 说明 `history: rewindable` 与 `fork: asOf` 组合的含义。

## 自测路径

```bash
# 单元测试
pnpm --dir packages/durable test

# 跑一个示例
node --conditions=source --experimental-strip-types packages/durable/test/examples/26-coding-agent.ts
```

## 延伸阅读

- `packages/durable/README.md` —— npm 包最贴地的一面。
- `packages/durable/docs/spec.md` —— 规范全文，读码准绳。
- `https://zhanghandong.github.io/pi-book/preface.html` —— pi 整体架构的姊妹书。
- `https://github.com/earendil-works/pi` —— 仓库本体。
