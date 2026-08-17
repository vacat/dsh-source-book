# 第 16 章 测试、不变量与质量门禁

> **本章目标**
> 1. 理解 dsh 的分层测试策略：unit / coverage / e2e / snapshot / web；
> 2. 理解运行时不变量（`ctx.invariants`）如何把契约变成可检查的断言；
> 3. 理解"验证世界，而不是自我报告"等测试哲学；
> 4. 理解质量门禁如何支撑一个大型 monorepo。

## 16.1 分层测试：证据要对得上表面

dsh 的测试不是"一个命令全跑"，而是**分层的**（`docs/testing.md` 的 Tiers）：

| 层 | 命令 | 作用 | 什么时候 |
|----|------|------|---------|
| **Unit** | `pnpm run test` | vitest 包内单测 + 仓库脚本单测 | 每次开发 |
| **Coverage** | `pnpm run test:coverage` | **门禁**：`packages/*/*/src` 逐文件 100% | CI 门禁 |
| **Real-API e2e** | `pnpm run test:e2e` | 有 key 的真实 API 测试 | 需要 `DEEPSEEK_API_KEY` |
| **Snapshot** | `pnpm run test:snapshot` | 无 key 的回放 vs 预期输出 | 模型/产品可见行为变化 |
| **Web snapshot** | `pnpm run test:web` | Chromium 浏览器回放对比 | Linux PR 门禁 |

**AGENTS.md 的指导原则**（读 testing.md 时的主心骨）：

> Match evidence to the surface: focused tests for behavior, snapshots for
> model or user output, doc-sync for docs, build/hygiene and built smokes for
> published paths, and real-API e2e for provider behavior.

**"证据要对得上表面"**：改行为用单测、改模型输出用快照、改文档用 doc-sync、
改发布路径用构建冒烟、改 provider 行为用真实 API。**别用一个层去验证
另一个层的改动的证据**。

## 16.2 Coverage 门禁的哲学

`test:coverage` 是 CI 的**真正门禁**（不是 `test`）。`docs/testing.md` 的原话：

> Line coverage is necessary, never sufficient — it proves lines ran, not that
> the feature works as shipped.

并且一个反直觉的观点：

> An uncovered line is often dead code the gate is correctly flagging for
> deletion, not a missing test to bolt on.

**100% 行覆盖的真意**不是"每行都被测"，而是"没有死代码"——一行永远跑不到，
往往意味着它该被删掉，而不是给它补个测试。**覆盖率门禁首先是代码卫生工具，
其次才是测试工具。**

## 16.3 快照测试：靠"请求可重建"吃饭

第 5 章说过：快照测试（snapshot）是"模型可见 ⟺ 可记录"的直接受益者。
`docs/testing.md` 的细节：

- **ACP snapshot**：启动真实的 automation-server 示例，回放录制的会话，
  diff 归一化的 JSON-RPC + 重新持久化的日志；
- **headless snapshot**：用显式示例组合跑 JSONL 测试驱动；
- 模型转录变化用 `test:snapshot:record`，回放输入仍有效用
  `test:snapshot:refresh`；**每次 JSONL 与预期输出的 diff 都要人工审**。

**为什么快照测试能"无 key 运行"**？因为它**回放**——输入来自日志（已录制），
模型输出用 `llm-replay` 适配器重放（第 8 章）。**没有可重建的日志，
这套测试基建根本不存在**。

## 16.4 运行时不变量：把契约变成断言

dsh 有一个**包自有不变量注册表**（`packages/runtime-diagnostics/invariants`，
`ctx.invariants`）。模块头：

> Configurable registry for package-owned runtime invariant contributions.
> Every workspace package registers checks from a `./invariant` companion;
> ordinary package entrypoints stay independent of diagnostics.

要点：

- **每个包注册检查**：每个 workspace 包有一个 `./invariant` 伴生文件，
  把自己的契约检查注册进来；
- **包名可过滤**：配置里有 `package_allowlist` / `package_blocklist`
  （正则），可以只跑某些包的检查；
- **失败即抛**：`InvariantFailure` 抛包归属的错误（"violated package
  contract"）。

**AGENTS.md 关于不变量的一般规则**（第 5 章提过）：

> **Runtime invariants assert owned relationships.** Check authoritative
> event streams or mutable data, not service or method presence, plugin
> metadata or effects, or fixed pure examples.

翻译：**运行时不变量断言"你拥有的关系"**——检查权威事件流或可变数据，
而不是"服务/方法是否存在""插件元数据/效果"或"固定的纯例子"。
**别用不变量检查"存在性"，要检查"关系"**（比如"这条消息的请求可从日志重建"）。

**为什么"模型可见 ⟺ 可记录"能成为不变量**：它有**可机械检查的对应物**
（从日志重建请求、逐字节对比），所以能写进 `ctx.invariants` 作为运行期断言，
而不是停留在文档里。

## 16.5 验证世界，而不是自我报告

`docs/testing.md` 有一条特别值得记的规则：

> Verify the world, not the self-report: an e2e assertion re-runs the command
> or re-reads the file externally; a keyword probe on the agent's own output
> lets a cheating agent pass.

**验证"世界"，不要验证"自我报告"**：断言应该**重跑命令/重读文件**来验证效果，
而不是在 Agent 自己的输出里搜关键词——否则一个"作弊的 Agent"（嘴上说改了、
实际没改）也能通过测试。**未触碰的文件要断言字节级一致。**

这条规则同样适用于不变量：检查**权威状态**（事件流、文件、磁盘），
而不是"它说自己做了"。

## 16.6 质量门禁全景

除了测试，dsh 还有一整套质量门禁（根 `package.json` 的 scripts）：

| 门禁 | 作用 |
|------|------|
| `pnpm run typecheck` | 全仓严格类型检查 |
| `pnpm run lint` | oxlint |
| `pnpm run duplication` | 跨文件克隆检测 |
| `pnpm run hygiene` | knip + publint + workspace 约束 + NodeNext 消费检查 |
| `pnpm run doc-sync` | 文档门禁（预算、i18n、目录） |
| `pnpm run website:build` | VitePress 构建（兼死链检查） |

**AGENTS.md 的检查哲学**（dsh-pre-push-checks 技能）：

> Never default to the full suite or repeat a passing check for commit or push.
> CI owns exhaustive coverage.

**别默认跑全量、别重复跑已通过的检查**——CI 拥有穷尽覆盖，本地只跑
"覆盖你这次改动的证据"。这既省时间，也让本地反馈聚焦。

## 16.7 测试文化的深层：把"工程质量"变成"可执行的纪律"

dsh 最值得学习的一点，是它**把工程纪律变成可机械检查的门禁**：

- 契约 → `ctx.invariants` 运行期断言；
- 模型可见输入 → 必须加会话事件（不变量抓住遗漏）；
- 快照 → 依赖可重建请求；
- 死代码 → 覆盖率门禁标出；
- 文档 → doc-sync 校验预算与同步。

**"纪律不是靠自觉，而是靠门禁"**——这是大型 monorepo 能长期健康运行的
根本原因，也是本书想传递的工程方法论。

## 16.8 本章小结

- 分层测试：unit / coverage（门禁）/ e2e / snapshot / web，证据对表面；
- 覆盖率 100% 首先是"无死代码"工具；
- 快照测试靠"请求可重建"吃饭；
- 运行时不变量断言"关系"，不检查"存在性"；
- 验证世界而非自我报告；未触碰文件断言字节一致；
- 质量门禁把纪律变成可执行检查。

## 动手练习

1. 打开 `docs/testing.md`，找出"真实入口路径"一节，理解为什么子进程测试
   要跑 built `lib/` 而不是 src。
2. 找一个包的 `src/invariant.ts`，读它注册了什么检查，判断它检查的是
   "关系"还是"存在性"。
3. 思考题：为什么覆盖率的定位是"发现死代码"而不是"保证正确"？
   （答案见附录 C。）

---

**下一章**：[第 17 章 配置、预设与组合](./ch17-config.md)
