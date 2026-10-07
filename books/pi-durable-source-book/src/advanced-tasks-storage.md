# 第 8 章 任务抽象与存储契约

pi-durable 完全不把"任务"当作一个仅用于记录的多字段结构体。`type.ts` 中的 `TaskRecord` 是一个细粒度分布式并发原语, 有着严格的生命周期: pending → running / waiting → completing → terminal。本章由两个层面拆开理解：任务状态机的生命周期，与跨三个后端一致的 `Storage` 契约。

## 8.1 任务状态机

文档中 `docs/spec.md` (第 5 节) 用一个非常克制的词汇表描述任务:

| 状态 | 含义 |
|------|------|
| `pending` | 已创建, 未经执行 |
| `running` | 有一个实现正在运行 phase 函数 |
| `waiting` | 运行完一个 phase, claim 了外部任务完结 |
| `completing` | 该任务的 outcome 已决定(成功或失败)但继续运行到其"owned children"全部结束, 结束后才迁到 terminal |
| `terminal` | 任务结束: 用 `outcome` 上的返回值或错误 |
| `abortRequested` | 是否已置中止标记, 状态由后续调度收敛 |
| `background` | 是否为 background 任务 |

派生自任务定义的类型:

```ts
type TaskOptions = {
  background?: boolean;
  ownership: TaskOwnership;
};
```

`ownership` 只有两个值：任务属于一个对话，或属于另一个任务（父子链）。运行期由 `harness/scheduler.ts` 负责.

## 8.2 `tx.task` vs `tx.setTask`: 语句级原子写

`Transaction.applyTx`（`session/transaction.ts`）把 task 当作对象重写而不是 diff：`createTask / setDelta` 都是"写入整条记录"。这个选择简化了：你 never need merge。整个任务 record 存在数据库里就是完整 JSON；不存在部分写。

```ts
const state = await tx.task(id);
// ... 做什么
// 完整 commit: pi-durable 写回整个 task record, 而不是打下补丁
tx.setTask({ ...state, state: { phase: "prepare", ... } });
```

## 8.3 Storage 契约

`src/types.ts` 里的 `Storage` 是整个 pi-durable 的后端契约, 是手册第 10 节原话的工程版本。关键操作:

- **读**: `conversation()` / `entry()` / `task()` / `submission()` / `findDocument()` / `document()` / `scanDocuments()`.
- **写**: `commit(StorageWrite[])`，原子批次。一次 commit 返回 `Seq`。
- **扫描**: 所有 scan 接成一个 `Page` / `Cursor` 双取页循环, 不做跨进程分页延迟。

```ts
// src/types.ts 里的 Storage 常见部分（语义重写以省篇幅）
type StorageWrite =
  | { type: "conversation"; value: ConversationRecord }
  | { type: "entry"; value: EntryRecord }
  | { type: "task"; value: TaskRecord<JsonValue, JsonValue, JsonValue> }
  | { type: "submission"; value: SubmissionRecord }
  | { type: "document.create"; record; content: Extract<DocumentContent,{kind:"base"}> }
  | { type: "document.copy"; record; source }
  | { type: "document.change"; id; content: DocumentContent }
  | { type: "document.retire"; id };
```

### 语义一: 提交无中间状态

`Session` 层拿到 `Storage.write` 的**阵列**（array of StorageWrite）。这个 array 是"**一批中的多个写同时成功或同时失败**"。若失败, 后端应返回 `StorageRejected` 类型错误——由 `types.ts` 统一约定。

### 语义二: Cursors 搬运

**Cursors is server-owned**—`cursor` 只能被某次同一后端 scan 返回再回传, pi-durable 不解释其内部结构。Cursors 有两个保证:

- **延续性**: 只能再用到与原 scan 相同的查询上。
- **一致性**: 相同 scan 的两个 cursor 在同一存储后端下互相兼容（同一点上开始扫描）。

### 语义三: 文档 incarnation 默认只读

`findDocument()` / `document()` 都只**返回”活跃 incarnation“**; retired incarnations 只能通过 `at: number(seq)` 式的点来取，若取不到则 `undefined`。从而 pi-durable **后端特征**：
`document(id, at)` 物化一个具体 incarnation, 而 `findDocument()` 解析逻辑地址 (kind/scope/key) 用 current 或历史点。

## 8.4 Forks: 语义中的"历史树"

`types.ts` 的 `ConversationRecord`:

```ts
type ConversationRecord = {
  id: ConversationId;
  parent?: { conversationId; at: EntryId };
  owner?: { conversationId; taskId };
};
```

`parent` 定义"历史 fork"; `owner` 是任务的归属记录。**fork 一个对话, 就是把 parent 指到一个 EntryId**: 子会话从这里"继承"前缀。

## 8.5 阅读清单

- `src/types.ts` — `TaskRecord` / `Storage` / `Cursor` / `Submission` 等类型定义常驻层。
- `docs/spec.md` 第 5、10 节 — 任务状态机与存储契约的规范原文。
- `src/harness/scheduler.ts` — 调度器和任务图索引。
- `src/session/transaction.ts` — `tx.task()` / `tx.setTask()` 语义。
