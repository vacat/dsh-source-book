# 第 17 章 配置、预设与组合

> **本章目标**
> 1. 理解 cordis.yml 如何"组合"出一个运行中的 Agent 产品；
> 2. 理解 `!!js`、overlay、config catalog 等加载细节；
> 3. 理解 preset（agent-preset / agent-preset-registry）：按会话组装不同的能力集；
> 4. 理解 settings / credentials 这两个面向用户的接缝。

## 17.1 配置是"组合"，不是"设置"

在 dsh 里，"配置"不是一坨参数，而是**组合出插件树**的手段。回顾第 1 章：
一个运行中的 dsh 是**按有序层组合出来的插件树**（profile → bundles → 补丁）。
`cordis.yml` 就是这个树的"编排文件"——每一条 entry 挂载一个插件、给它的
`config` 传参、决定它是否 `disabled`。

```yaml
# 一个极简 cordis.yml 片段（示意）
plugins:
  - id: llm
    config:
      provider: deepseek
      model: deepseek-chat
  - id: tool-bash
  - id: my-preset
```

**AGENTS.md 的加载约定**：

> cordis.yml allows `!!js` (never `!js`) under plugin `config` and entry
> `disabled`; other metadata stays literal, so conditional composition also
> uses overlays.

- `config` 与 `disabled` 允许 `!!js` 表达式（注意是**双感叹号**）；
- 其它元数据保持**字面量**；
- 环境要选择插件时用 **overlay（覆盖层）**，而不是在 yml 里写死条件。

## 17.2 config catalog：生成式配置参考

`docs/config-catalog.md` 是**从源码自动生成**的配置目录
（`scripts/gen-config-catalog.ts`），列出了每个 cordis.yml entry 可设置的
`config` 字段。注意它的三个保证：

1. **逐字粘贴声明的类型**：每个插件的 `apply`/构造器收到的 config 类型
   原样贴出（含 JSDoc）；
2. **schema 与声明互查**：运行时 schemastery schema 与声明的类型
   **交叉校验**——"loader 能接受的字段，必须能在声明类型上找到"，
   避免"配置目录里藏着 loader 接受的隐藏字段"；
3. **`Requires:` 行**列出插件 inject 的服务——**配置树必须同时加载这些
   服务的 provider**。

**文档由代码生成 + 校验**：这是 dsh 的一贯风格（架构图、事件图、配置目录
都从代码抽取，保证永不过时）。

## 17.3 preset：按会话组装能力

`packages/preset/agent-preset`（在 Cordis YAML 里声明一份能力组合）和
`packages/preset/agent-preset-registry`（注册表服务 `ctx.agentPresets`）
一起实现**"按会话组合能力"**。
回顾 `docs/architecture.md` 的"Where new behavior goes"：

> Give one session a different capability set | compose an agent preset; a
> service row there needs an `isolate` realm

**preset 是"会话级组合"**：不是全局装什么就是什么，而是**每个会话可以
指定一组不同的能力集**（挂不同的插件、注册不同的工具、注入不同的人设）。
这就是第 13 章 scope 在"产品组合"层的应用——**preset 决定了这个会话的
插件树上多挂哪些分支**。

`isolate` realm 是一个细节：preset 里的服务行如果声明 `isolate`，
它的服务实例被隔离到该 preset 的作用域，**避免跨 preset 共享状态**。

## 17.4 settings 与 credentials：两个用户接缝

- **`ctx.settings`**（`packages/settings/settings`）：
  用户设置接缝——"用户想改什么"（主题、偏好、模型默认值）通过它读写；
- **`ctx.credentials`**（`packages/credentials/credentials` + `credentials-local`）：
  凭据接缝——API key 等敏感信息，不写死在配置里，通过凭据提供方读取
  （env、.env、用户环境层）。AGENTS.md 的 secrets 策略：
  > Never commit credentials. CI e2e skips without a key.

**settings 与 credentials 的区别**：settings 是"非敏感的偏好"，credentials 是
"敏感的密钥"。dsh 把它们做成**独立接缝**，让"凭据从哪来"（env / .env /
凭据管理器）可以替换，而不污染用户设置。

## 17.5 组合的分层：从 bundle 到 preset

把本章与第 1 章连起来，得到 dsh 的完整组合分层：

| 层 | 控制什么 | 谁决定 |
|----|---------|--------|
| **bundle** | 一个可分发的能力包（base / web-app / headless） | 打包者 |
| **profile** | 一个产品形态的 bundle 列表 + 用户补丁 | 部署者/用户 |
| **preset** | 一个会话的能力集（挂哪些插件、注入什么人设） | 会话创建方 |
| **scope** | 一次注册的可见范围（第 13 章） | 插件作者 |

**bundle 决定"产品装了什么"，preset 决定"这个会话额外有什么"，scope 决定
"这次注册给谁看"。** 三个层次各有归属，不互相越权。

## 17.6 配置的"大声失败"原则

AGENTS.md 有一条规则：

> **Misconfiguration fails loud** at load when self-contained, otherwise at
> the earliest resolvable point; never silently skip a missing referent.

**错误配置要大声失败**：自包含的问题在加载时就报错，否则在最早可解析的
点报错；**绝不静默跳过缺失的引用**。这条规则让"配置错了"在第一时间暴露，
而不是让 Agent 跑着跑着出现诡异行为。

## 17.7 本章小结

- 配置 = 组合插件树（profile / bundle / 补丁），不是参数表；
- `!!js`（双感叹号）用于 config/disabled，条件组合用 overlay；
- config catalog 由代码生成 + schema 互查，保证"配置目录不藏字段"；
- preset 按会话组装能力集，`isolate` realm 隔离服务实例；
- settings（偏好）与 credentials（密钥）是独立接缝；
- 错误配置大声失败，绝不静默跳过。

## 动手练习

1. 打开 `docs/config-catalog.md`，找一个你熟悉的包（如 `dsh-llm` 或
   `dsh-tool-bash`），读它的 config 类型与 `Requires:` 行。
2. 找一个 preset（`packages/preset/agent-preset-registry` 或示例），理解它是怎么
   "按会话组装"插件的。
3. 思考题：为什么"配置错误要大声失败"而不是"尽量容错继续跑"？
   （答案见附录 C。）

---

**下一章**：[第 18 章 动手：给 dsh 加一个新工具](./ch18-extension.md)
第四部分到此结束。接下来是全书最"动手"的部分——把你的知识用到真实 dsh 里。
