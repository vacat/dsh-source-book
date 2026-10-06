# 第 18 章 动手：给 dsh 加一个新工具

> **本章目标**
> 1. 掌握"最小工具形态"：`defineTool` + `ctx.tools.register`；
> 2. 吃透 `execute()` 契约的规则（校验、身份、返回值、signal）；
> 3. 理解 UI 呈现（render intent）是工具设计的一部分；
> 4. 用 dsh 的验证方法收尾（测试 + 快照）。

## 18.1 最小工具形态

`docs/cookbook/adding-a-tool.md` 给了一个完整的"最小工具"。核心就这几步：
**导出插件 → 注入 `tools` → 注册工具**。

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',          // 模型看到的
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                     // 默认可选
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args 由 schema 推导类型：{ path: string; limit?: number }
      // exec 携带不可变身份 + token；signal 是操作字段
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

**观察三个关键点**：

1. **注册即效果**：`ctx.tools.register` 注册的工具，在插件 fiber 被卸载时
   自动注销（第 2 章"注册即副作用"）——不需要手动清理；
2. **schema 自动进入系统提示组装**：注册后，工具的 schema 会自动出现在
   模型可见的工具清单里（第 13 章）——**你不需要手动把它塞进提示**；
3. **`args` 是类型推导的**：`defineTool` 会根据 `parameters` 推导出
   `execute` 的参数类型，编译期就保证类型正确。

## 18.2 `execute()` 契约的规则

cookbook 列出几条"硬规则"，每一条都有背后的工程原因：

| 规则 | 为什么 |
|------|--------|
| **参数自动校验**：`defineTool` 在 `execute` 前校验模型生成的 arguments | 模型的输出不可信，必须在边界校验（`validateArgs()`，见 `packages/core/tools/README.md` 与 `docs/cookbook/adding-a-tool.md`） |
| **注册借用只读定义**：注册后不要改 schema 或回调 | 类型化同进程贡献不是序列化边界；要换工具就"卸载旧效果 + 注册新效果" |
| **执行身份受保护**：`exec.token` 不透明；`callId/name/arguments/agent/token/signal` 不可变 | 防止策略层篡改执行身份 |
| **返回一个规范 JSON 值**：`output.schema` 声明，`execute` 只返回值 | 让返回值可序列化、可校验、可做 UI |
| **抛错 / 返回非法值 = `isError`** | 把"失败"变成结构化的结果，而不是崩溃 |
| **尊重 `exec.signal`**：信号触发时取消在途工作 | 支持取消（第 7 章的中止语义） |
| **用 `exec.agent` 做异步通知**：`agent.inject(...)` 追加到下次请求 | 模型可见上下文要进会话（第 5 章），但不是唤醒 |

**最值得反复读的两条**：

- **"模型/UI 分层"**：`output.render` 管"模型看到的文本"，UI 卡片是**另一个
  关注点**，通过 `presentCall` / `presentResult` 纯函数声明。**UI 专用的格式
  （```console 块、diff、相对路径）绝不该塞进模型结果里**。
- **"呈现必须是纯函数"**：presentation 在**流式播放和日志回放**上都会跑，
  所以**不能有 I/O、不能读会话状态、不能有随机**。diff 只能从 args 推导
  （`write` 用 `oldText: null`，因为调用时没有旧文件内容）。**回放安全**——
  这是"模型可见 ⟺ 可记录"在 UI 层的延伸。

## 18.3 UI 呈现：card 与 render intent

AGENTS.md 有一条规则：

> **A tool's UI render intent is part of its design**, decided up front
> (`generic`/`terminal`/`diff`, `locations`); presentation methods are pure
> functions of `args`.

工具的 UI 呈现**在设计阶段就要决定**，用 card 标记表达：

| card | 用途 | 例子 |
|------|------|------|
| `generic` | 默认卡片，可带 `locations`（文件定位） | 读文件、搜索 |
| `terminal` | 这个调用就是一条 shell 命令 | tool-bash |
| `diff` | 创建/修改文件，渲染内联 diff | tool-fs 的 write/edit |
| `search` | 发现结果（grep/glob） | tool-fs-search |
| `web` | 网页检索结果 | tool-web |

**"中性词汇住在 dsh-tools，工具从不 import UI 或传输类型"**——host/client
各自把 card 映射成自己的视图。这让同一工具在不同 UI（终端、浏览器、IDE）
都能呈现。

## 18.4 长任务与后台作业

cookbook 的 "Long-running work" 一节：工具要支持后台运行时，用
`run_in_background`（由 producer 配置门控）+ `ctx.jobs.start({ kind, label,
owner: exec.agent, run })`。**注意 `owner: exec.agent`**——后台作业的归属
是发起它的 Agent（呼应第 6 章 Initiating Agent）。

## 18.5 验证：工具不是写完就完

cookbook 的 "Verification" 一节要求工具开发用 dsh 的完整验证链：

1. **单元测试**：覆盖参数校验、错误路径、事件顺序（第 16 章）；
2. **快照**：模型可见输出变化要加无 key 快照（回放，不是 mock）；
3. **真实组合测试**：产品可见插件需要"非单元的真实组合"测试——boot 真实的
   cordis.yml，而不是手拼 `ctx.plugin(...)`。

**"别用 mock 顶替真实组合"**——mock 只能证明"桥能传字节"，不能证明
"上线的工具行为符合断言"。

## 18.6 动手作业

现在给你一个真实练习：**给 dsh 加一个 `word_count` 工具**（统计文本文件
的词数/行数）。

1. 照 18.1 的最小形态，写 `packages/<你的目录>/word-count/src/index.ts`；
2. 设计 `parameters`（path 必填）与 `output.schema`（对象：{words, lines}）；
3. 实现 `execute`（用 `readFile` + 简单统计），尊重 `exec.signal`；
4. 加一个 `presentCall` 返回 `{ card: 'generic', locations: [{ path }] }`；
5. 写单元测试（读存在文件、读不存在文件抛 isError）；
6. 用第 16 章的"真实组合测试"思想，通过 cordis.yml 挂载验证模型能看见它。

完成标准：模型能调用它、结果进日志（可回放）、卸载后工具消失。

## 18.7 本章小结

- 最小工具 = `defineTool` + `ctx.tools.register`，注册即效果；
- `execute()` 契约：自动校验、身份保护、规范返回值、尊重 signal；
- UI 呈现是设计的一部分：card / render intent，且必须是纯函数（回放安全）；
- 长任务用 `ctx.jobs.start`，归属发起 Agent；
- 验证用"单测 + 快照 + 真实组合"，别用 mock 顶替。

## 动手练习

1. 读 `docs/cookbook/adding-a-tool.md` 全文，特别是 "Rules of the execute()
   contract" 一节。
2. 读 `packages/shell/tool-bash`（生产级三包示例），看它如何拆
   Definition / Provider / Consumer。
3. 思考题：为什么 `output.render`（模型看到的）和 `presentResult`（UI 卡片）
   必须分开？混在一起会破坏什么？（答案见附录 C。）

---

**下一章**：[第 19 章 贡献与演进：Agent Notes 与自修改](./ch19-contribute.md)
