# 附录 C 服务（ctx.*）清单

> 常用 `ctx.*` 服务速查。完整服务图见 `docs/capability-seams.md`
>（由代码自动生成）。按"脊柱 / 能力 / 产品"分组。

## 核心脊柱（core）

| 服务 | 拥有什么 |
|------|---------|
| `ctx.sessions` | 会话日志 + 内存 store（事件溯源） |
| `ctx.systemPrompt` | 系统提示分片组装 |
| `ctx.tools` | 工具注册表 + 执行流水线 |
| `ctx.agents` | Agent 注册表 + `agent/*` 事件 |
| `ctx.agentLoop` | 默认驱动（ReactLoopAgent） |
| `ctx.agentDefaultModel` | 默认模型选择 |
| `ctx.agentPresets` | 按会话组装能力（preset） |

## LLM 与模型

| 服务 | 拥有什么 |
|------|---------|
| `ctx.llm` | LLM 适配器注册表 + 流式调用 |
| `ctx.tokenMeter` | 回放 token 计量 |
| `ctx.credentials` | 凭据接缝（env / .env） |
| `ctx.settings` | 用户设置接缝 |

## 执行世界

| 服务 | 拥有什么 |
|------|---------|
| `ctx.subprocess` | 子进程接缝 |
| `ctx.shell` | bash 执行接缝 |
| `ctx.shellEnv` | 托管 bash 环境 |
| `ctx.terminals` | 持久 PTY 会话 |
| `ctx.sandbox` | 进程沙箱接缝 |
| `ctx.sandboxPolicy` | 沙箱策略归属方 |
| `ctx.fs` | 文件系统接缝 |
| `ctx.ptcRuntime` | PTC 代码执行接缝（模型写程序、调用宿主绑定） |

## 更多能力

| 服务 | 拥有什么 |
|------|---------|
| `ctx.web` | 网页访问（search/fetch） |
| `ctx.subagents` | 子代理接缝（多 provider） |
| `ctx.skills` | 技能注册表 |
| `ctx.workflowEngine` | 工作流脚本引擎 |
| `ctx.lsp` | 语言服务器导航 |
| `ctx.jobs` | 后台作业注册表 |
| `ctx.goals` | 会话目标域 |
| `ctx.userQuestions` | 人类问答接缝 |

## 交互与安全

| 服务 | 拥有什么 |
|------|---------|
| `ctx.approval` | 审批接缝 |
| `ctx.permissionPresets` | 权限预设（sandbox+approval 旋钮） |
| `ctx.commands` | 人类命令注册表 |

## 存储与持久化

| 服务 | 拥有什么 |
|------|---------|
| `ctx.sessionPersistence` | 会话持久化接缝（当前 provider：jsonl） |
| `ctx.storage` | 非会话存储枢纽 |
| `ctx.storageDomain` | 领域数据设施 |
| `ctx.spillStore` | 溢出存储接缝 |
| `ctx.sessionQuery` | 会话读取/追踪/搜索 |
| `ctx.sessionTitle` | 会话标题生成 |
| `ctx.sessionTelemetry` | 会话遥测 |
| `ctx.attachments` | 持久附件存储 |

## 工程

| 服务 | 拥有什么 |
|------|---------|
| `ctx.typert` | 运行时类型注册表 |
| `ctx.typertGateway` | Typert 调用网关 |
| `ctx.webServer` | HTTP 路由注册 |
| `ctx.clientModules` | 客户端插件图宿主 |
| `ctx.dynamicCordisRunner` | 动态 Cordis 包宿主（自修改） |
