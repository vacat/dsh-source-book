# 《dsh 源码进阶：读懂 DeepSeek Harness》

> **面向已掌握 Agent 基础概念的工程师，深入 DeepSeek Harness（dsh）源码，逐行读懂一个"一切皆插件"的企业级 Agent 框架。**

## 关于本书

这是《Agent 技术实战：从零读懂三大 Agent 代码库》（agent-book）的进阶姊妹篇。agent-book 教你理解"推理循环"这个通用骨架；本书则把镜头对准 **dsh（DeepSeek Harness）** 一家，深挖它区别于其它 Agent 框架的工程机制。

### 本书回答的问题

- 什么是"一切皆插件"？没有特权核心，每个能力都能从配置替换？
- 事件溯源（event sourcing）的会话日志如何做到"模型可见 ⟺ 可记录"？
- 推理循环如何被拆成 turn/step、收件箱、状态机、瀑布？
- 能力接缝（capability seam）的三角色如何让"换一个 provider 就换整个产品"？
- scope、沙箱、审批、持久化、压缩、测试不变量……这些企业级能力如何落地？

### 特点

- **源码精读**：第 7、8、9 章对 agent-loop、LLM 层、工具调度逐行拆解
- **架构优先**：以 `docs/architecture.md`、`docs/subsystems/`、Agent Notes 为纲
- **由浅入深**：总览 → 核心机制 → 能力接缝 → 工程与质量 → 实战，5 大部分 19 章
- **工程文化**：讲解 dsh 的 Agent Notes（每次架构决策的记录）与质量门禁

### 读者要求

- 已读过 agent-book，或已掌握"推理循环 / 工具 / 会话"等基本概念
- 会读 TypeScript
- 不需要大模型内部知识

### 代码版本

本书基于 dsh 仓库 commit `49120665f`（2026-09-26）。dsh 仓库：<https://github.com/deepseek-ai/deepseek-harness>

### 全书结构

| 部分 | 章节 | 主题 |
|------|------|------|
| 第一部分 总览与地基 | 第 1-3 章 | 全景、Cordis 插件框架、事件系统 |
| 第二部分 核心机制 | 第 4-9 章 | 会话日志、请求可重建、Agent、agent-loop、LLM、工具 |
| 第三部分 能力接缝 | 第 10-13 章 | 接缝模式、shell、web/subagent/skill/MCP、system-prompt 与 scope |
| 第四部分 工程与质量 | 第 14-17 章 | 持久化与压缩、安全审批、测试不变量、配置预设 |
| 第五部分 实战 | 第 18-19 章 | 扩展 dsh、贡献与演进 |
| 附录 | A-D | 术语表、事件速查、服务清单、延伸资源 |

## 如何阅读

1. 网页版：`mdbook serve --open`（http://localhost:3000）
2. 从 `src/SUMMARY.md` 查看完整目录
3. 建议先读第 1 章建立全景，再按顺序深入

## 本地构建（mdBook）

```bash
brew install mdbook
cargo install mdbook-mermaid
mdbook-mermaid install
mdbook build          # 输出 book/
mdbook serve --open   # 本地预览
```

## 目录结构

```
├── book.toml              # mdBook 配置
├── src/                   # 书籍源码
│   ├── SUMMARY.md
│   ├── preface.md
│   ├── ch01-*.md ~ ch19-*.md
│   └── appendix-*.md
├── theme/custom.css       # 中文排版样式
└── .github/workflows/deploy.yml  # mdBook 部署（如需 GitHub Pages）
```

## 配套书籍

- **agent-book**：《Agent 技术实战：从零读懂三大 Agent 代码库》——通用 Agent 入门
- **llm-inference-book**：《大模型推理技术详解》——大模型推理源码解读

## 许可

本书原创文字以 CC BY-NC-SA 4.0 提供。dsh 源码版权归 DeepSeek，遵循其仓库许可。
