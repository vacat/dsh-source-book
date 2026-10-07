# 第 9 章 扩展机制:扩展、钩子、工具

pi-durable 的扩展机制是它们"在自己的容器里造自己的 Agent"最主要入口。`defineExtension()` + `defineTool()` + `hook()` + `section()` + `wrapTool()` 五个函数共同组成一个完整的扩展DSL。本章过一遍这五个函数的语义链。

## 9.1 `defineExtension`:包裹五类插件

README 明示了扩展可以携带的所有部分:

```ts
// 伪签名
const myExtension = defineExtension({
  name: "my-extension",
  tools: [...],      // 产 Tool
  sections: [...],   // 系统提示词段落
  hooks: [...],      // beforeTool / afterTool 等
  wraps: [...],      // 按名字给已定义 tool/section 做装饰
  tasks: [...],      // 自定义任务
});
```

`registry.install(ext)` 不会 modify 其他字段;扩展里塞进 registry 的是这个对象, **`pi.agent` 文档只存 extension/plugin 的 name**。

### 同名覆盖

后装的扩展同名"name 覆盖前装", `registry.uninstall(ext)` 按 name 移除。README 明示"same name replaces"。

## 9.2 `defineTool`:工具定义与输入校验

`defineTool()` 需要提供 `name`, `description`, `parameters`(TypeBox schema), `execute`。可选字段:

- `replay`: `"safe"` / `"unsafe"`。默认 unsafe。`safe` 表示"重复执行无副作用", 崩溃恢复时 pi-durable 可直接重入 execute。

README 例子输出了工具定义精义:

```ts
const Subagent: Extension = defineExtension({
  name: "subagent",
  tools: [
    defineTool({
      name: "subagent", description: "Delegate...",
      parameters: Type.Object({task: Type.String()}),
      // ...
      replay: "safe", // a rerun after a crash finds the same child and submission
      execute: async (args, api, context) => {
        const child = await api.commit(async (tx) => {
          const existing = (await tx.scanConversations({...}, 1)).items[0];
          if (existing !== undefined) return existing.id;
          const created = await tx.createConversation({ownership: {kind: "task", taskId: api.taskId}});
          return created.id;
        }, context);
        // ...
      },
    }),
  ],
});
```

这段代码的含义是清晰: **`api.commit()` 内做"fork 语义下的子对话创建"**——把一次性"createConversation"行为以事务表达, 并在 crash 后重放"找老 child"逻辑。这是工具中带副作用时的标准做法。

## 9.3 `defineSection`:提示词插件

`pi.system` entry 提供模型提示词的“分节补丁”。README 原话: "sections is an ordered named patch"。`sections` 的 render:

```ts
section("cwd", (input) => input.env?.cwd); // 渲染为 <cwd>...</cwd>; undefined 消息不存在
```

一个 section 是一个命名键值 (key 加 tag 包裹)。两个 section 同名时, **后写入的值替换前值**, null 删除值。README 也写明: "A conversation's `instructions` render last, as the section `instructions`."

## 9.4 Hook 语义:十六种钩子点

`harness/hooks.ts` 定义了扩展钩子调和点:

```ts
hook(ToolTask, { beforeTool: (call) => ... })
```

`hook()` 的**第一个参数**到底是哪一个内置任务? README 提到四对 hook 点:

- **Generation**: `beforeRequest`, `afterResponse`, `onYield`, `afterTools`
- **Tool**: `beforeTool`, `afterTool` (以及 wrapTool 例外)

hook 函数返回值决定行为 (以 `beforeTool` 为例): 返回 `undefined` 走默认处理；返回 `{ block: ... }` 收到"抗拒该调用", 合同化的 message 会作为 tool-result 提交给模型。

## 9.5 wrapTool / wrapSection:在名字层做装饰

README 例子:

```ts
const Venv = defineExtension({ name: "venv",
  tools: [createBashTool({ commandPrefix: "source .venv/bin/activate" })] });
const Timing = defineExtension({ name: "timing",
  wraps: [wrapTool(createBashTool(), (bash) => ({...bash, execute: (args, api, ctx) => timed(...)}) )] });
```

`wrapTool(createBashTool(), wrapper)` 就是在"名字为 `bash` 的 tool"上增加一层**无副作用 wrapper**, 用于计时、input 记录等。`wrapSection` 同理。

## 9.6 `wrapTool` 和 `wrapSection` 不是 override

一句话: `wrap` 是**authenticated** 装饰, **不改变工具的身份** (name 不变); override (同名覆盖) 才改变身份。使用场景: 如果你的扩展中有一类“需要一个更小的ts, 可以 wrap 一些内建工具而不令用自己的 name。

## 9.7 `defineDoc`:扩展自带业务数据

README 例子:

```ts
const Todos = defineDoc<{items: string[]}>({
  kind: "app.todos",
  version: 1,
  scope: "conversation",
  history: "latest", // 或 "rewindable"
  fork: "initial",
  initial: () => ({items: []}),
});
```

`history: "rewindable"` + `fork: "asOf"` 组合, 让 fork 时看到**父对话在 fork 点前面的文档值**, 而不是"子对话首次读到时的 active value"。

## 9.8 Reading list

- `src/harness/define.ts` — `defineExtension` / `defineTool` / `hook` / `section` / `wrapTool` / `wrapSection` 的签名和注释。
- `src/harness/hooks.ts`—扩展 hook 的实现。
- `docs/spec.md` 第 7 节 — 扩展, 钩子, 工具与 system prompt 规范。
