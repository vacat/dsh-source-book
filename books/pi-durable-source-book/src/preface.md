# 关于本书

这不是 pi-durable 的 API 手册，也不是一篇"怎么装怎么跑"的教程。它是一本 **面向源码的导览**：假设你已经写过或读过若干 Agent 循环，现在想把 pi-durable 这个包本身掰开，看清它是如何让"Agent 会话"在一个进程意外消失后还能从磁盘原样续上的。

## 适合谁

- 有工程经验、想在自己的 Agent 产品中借助 pi-durable 作为持久化底座的开发者；
- 想给 pi 生态贡献源码或写 extension 的人；
- 需要评估"是否有必要把 Agent 循环搬到持久化底座上"的技术负责人。

## 阅读前提

- TypeScript：能读懂类型与 `async/await` 的混合代码即可；
- 对 LLM 的 tool call / streaming 有基本认知；
- 不需要用过 pi 本身，也不需要懂 SQLite 内核。

## 本书怎么读

三条路径：

- **架构师**：第 1 章 → 第 3 章（Session 层）→ 第 4 章（Store 层）→ 第 8 章（任务与契约）→ 第 9 章；
- **开发者（写扩展或换后端）**：第 2 章 → 第 5 章 → 第 6 章 → 第 7 章 → 附录 B；
- **完整阅读**：按第 1 章到附录 B 顺序读完。

## 版本基线

本书分析基于 `earendil-works/pi` 仓库 `main`（对应 npm 包 `@earendil-works/pi-durable` v1.0.4（最新发布））的本地副本。所有文件路径与行号指向该基线，后读版本若有改动请以当时仓库为准。

## 排版约定

- 源码引用格式如 `packages/durable/src/session/session.ts:80`，指向 pi 仓库内的位置。
- 内建对象名（`Harness`、`SessionImpl`、`ToolTask`）一律用行内代码。
- 每章末尾列出"Reading list"，把当章引用的核心文件索引在一处，便于下一章开始前定位代码。
