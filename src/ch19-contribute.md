# 第 19 章 贡献与演进：Agent Notes 与自修改

> **本章目标**
> 1. 理解 Agent Notes：dsh 如何把"为什么这样设计"变成仓库资产；
> 2. 掌握 Agent Note 的分类、格式与归档规则；
> 3. 理解"决策驱动阅读"——用 Agent Notes 读复杂代码；
> 4. 了解 dsh 的自修改能力（demo:cordis）与贡献流程。

## 19.1 什么是 Agent Notes

dsh 的 `.agents/notes/` 目录是一个**决策记录库**：每一次非平凡改动，都要
在同一 PR 里新增或更新至少一篇 **Agent Note**（`docs/AGENTS.md` 与
`.agents/notes/README.md` 的硬规则）。

> Every non-trivial change MUST add or update at least one Agent Note in the
> same PR.

**为什么**：大部分代码库的注释解释"这段代码干什么"，但很少解释
"为什么这么做、为什么不用另一个方案"。Agent Notes 专门记录后者——
**问题 → 决策 → 备选方案 → 后果**。它把"设计意图"变成可检索的仓库资产。

## 19.2 三种生命周期与文件格式

Agent Note 按决策状态分三个目录：

| 目录 | 状态 | 内容 |
|------|------|------|
| `proposed/` | 提议 | Problem → Proposal → Alternatives → Acceptance criteria → Risks |
| `implemented/` | 已实现 | Problem → Decision → Alternatives → Consequences |
| `rejected/` | 已拒绝 | 冻结的提议 + Status 行写原因 |

**文件格式有硬性规定**（`pnpm run verify-agent-note-format` 门禁）：

- 前三行固定为 `# Agent Note: <标题>`、`Status: <状态>`、空行；
- **`## Alternatives considered` 是强制的**：
  > A decision recorded without what it beat invites re-litigation.

  **没有记录"它打败了什么"的决策，会招致反复争论**——每个方案都要记录
  "为什么输"；
- `implemented/` 里**禁止**出现 Proposal 时代的标题
  （`## Proposal`、`## Plan`、`## Migration plan`、`## Acceptance criteria`）——
  已实现就要用现在时描述"已经发生的事实"，不说"打算"。

## 19.3 归档：冻结，而不是编辑

`README.md` 的归档规则（配合 `dsh-archive-agent-notes` 技能）：

> Once sealed, every archived triplet is permanently frozen. Do not edit,
> translate, reformat, update, move, or delete it, and do not treat it as
> authority for current behavior.

**归档的笔记永久冻结**：不能改、不能当作当前行为依据。已归档决策过期了？
写一篇**新的**实现笔记来取代它，并在两者之间交叉链接。**这防止"历史的
理由被改写"**——这也是 dsh 的 AGENTS.md 反复强调
（"Archived notes are frozen: never edit or treat them as current authority"）。

## 19.4 决策驱动阅读：本书的方法论

读者朋友，这是全书最实用的一节。**当你读 dsh 代码时，不要从函数开头硬啃，
先找它对应的 Agent Note。** 流程：

```mermaid
flowchart LR
    FILE[遇到一个复杂模块] --> NOTE[查 .agents/notes 找相关 Note]
    NOTE --> WHY[读 Problem + Decision + Alternatives]
    WHY --> CODE[带着"为什么"回到代码]
    CODE --> VERIFY[对照 Consequences 验证理解]
```

以第 4 章为例：你读 `deriveMessages()` 时，先读
`2026-06-11-event-sourced-sessions`，它告诉你"为什么用事件溯源而不是
可变数组"（"state and log can diverge; with event-sourcing the log IS the
state, so divergence is structurally impossible"）。**带着这个"为什么"
读代码，每一行都有了意义。**

**这就是 dsh 邀请所有贡献者使用的工作方式**——也是这本书想传递给你的
"读大型代码库"的钥匙。

## 19.5 自修改：Agent 挂载自己的插件

dsh 有一个"自我指涉"的玩法：**Agent 能检查并挂载它自己的插件**
（`demo:cordis`）。看 `scripts/demo-cordis.mjs` 的注释：

> Boot the self-referential Cordis tools under Web or ACP ... This is a
> repository demo wrapper, not a product CLI feature.

```sh
pnpm run demo:cordis          # Web 界面，port 3081
pnpm run demo:cordis acp      # ACP 模式
```

实现落在 `packages/extensions/tool-cordis`（self-referential Cordis 工具）
与 `packages/extensions/cordis-host-runner`（动态 Cordis 包宿主运行器）。
它展示了一个前沿能力：**Agent 在运行中把自己的一部分（一个工具、一个插件）
换掉**——这依托于第 2 章的"注册即副作用、挂上能卸"。**自修改不是魔法，
是"可逆效果"叠加"动态宿主"的自然结果。**

## 19.6 贡献流程：怎么进入这个系统

1. **读约定**：`CONTRIBUTING.md`、根 `AGENTS.md`、`docs/AGENTS.md`；
2. **定位扩展点**：用第 1 章"Where new behavior goes"表，找到你的改动
   该挂在哪；
3. **写 Agent Note**：非平凡改动在同一 PR 里写 Note（19.2 的格式）；
4. **过质量门禁**：第 16 章的门禁（typecheck / lint / 针对性测试 / snapshot /
   doc-sync）；
5. **本地只跑相关检查**：别全量跑、别重复跑已通过的检查——CI 拥有穷尽覆盖。

**AGENTS.md 的提交纪律**（dsh-pre-push-checks 技能）：
> Report only commands run. After `gh stack sync`, validate immediately;
> do not merge before checks pass.

## 19.7 本书收尾：dsh 给你留下的方法论

读完这本书，你带走的应该不只是"dsh 的架构"，而是三件可迁移的东西：

1. **一切皆插件 + 注册即副作用**：任何大型可扩展系统都该考虑
   "无特权核心 + 可逆注册"；
2. **事件溯源 + 模型可见 ⟺ 可记录**：任何 Agent 系统都该保证
   "模型看到的一切可重建"——这是调试、测试、审计的地基；
3. **决策记录（Agent Notes）+ 门禁化纪律**：把"为什么"写进仓库、
   把"纪律"变成可机械检查的门禁——这是大型 monorepo 长期健康的关键。

## 19.8 本章小结

- Agent Notes 记录"问题→决策→备选→后果"，非平凡改动必写；
- 三种生命周期（proposed/implemented/rejected）+ 强制 Alternatives 节；
- 归档笔记永久冻结，过期决策用新笔记取代；
- 决策驱动阅读：先读 Note 再读代码；
- 自修改 = 可逆效果 + 动态宿主（demo:cordis）；
- 贡献 = 定位扩展点 + 写 Note + 过门禁 + 只跑相关检查。

## 动手练习

1. 选一篇你在这本书里读过的架构 Note（如 `event-sourced-sessions`），
   把它的 Problem / Decision / Alternatives / Consequences 四段各用一句话
   复述。
2. 用"决策驱动阅读"读一个新的模块：先搜 Note，再读代码，验证你的理解
   与 Consequences 一致。
3. 思考题：为什么 `implemented/` 笔记禁止出现 `## Proposal` 这种"未来式"
   标题？这反映了一种什么工程价值观？（答案见附录 C。）

---

**接下来**：[附录 A 术语表](./appendix-a-glossary.md)
恭喜你读完全书。附录里有术语表、事件速查、服务清单和延伸资源，随时查阅。
