# 第 4 章 Store 层：Memory、SQLite、JSONL

pi-durable 把"durable"的落地托付给一组 `Storage` 后端。本章沿 `src/storage/*` 目录逐个看三个后端怎么把 `StorageWrite` 从一个接口稳定成物理介质上的字节，以用什么语义锁定"提交点"。

## 4.1 storage 目录的三套后端

```
src/storage/
├── memory.ts     内存实现，无锁共享
├── sqlite/       SQLite 后端（含 migrations）
│   ├── storage.ts       Storage 接口实现
│   ├── migrations.ts    schema 演化
│   └── node.ts          node:sqlite 适配器
└── jsonl/        JSONL 追加日志
    ├── storage.ts       Storage 接口实现
    └── node.ts          node:fs 适配器
```

三者的分工，按 README 一段话简述：

- Memory：纯内存，`Map` + 链表，进程崩溃即丢，适合单测。
- SQLite："一个数据库文件，按 WAL 模式重放崩溃（最新提交在崩溃时可能丢失）"。
- JSONL："追加日志文件，每个 commit 追加一条 marker，再把 main.jsonl 里的 append 后端真正落盘（可选 fsync）"。

## 4.2 MemoryStorage：一套 `Map` 实现整个契约

`memory.ts` 用一组 `Map` 记录五类表：`recordTypes`（id → 表名，帮助后续 query 定向）、`conversations`、`entries`、`tasks`、`submissions`，外加一个 `DocumentAddressIndex`（把文档家族和单例查询合并到一张 `Map`）。

读请求与写请求都是"本地读取后做并行映射"；没有任何锁。为了模拟"跨进程孤岛"，`MemoryStorage` 是"detached"的语义——所有读都是**浅拷贝**，`#guarded` 会保证调用方拿到的对象不会不经意地改写底层数据：

```cpp
// 注释示意，实际实现为 JS
Documents table: 当前值的拷贝……
```

这套"拷读"行为，是 `testing/runner.ts` 里进程隔离测试的基础，也是 Durable Object 等无锁后端能复用同一接口的关键。

## 4.3 SQLite 后端和 migrations

`sqlite/migrations.ts` 只有 schema version 1（`SQLITE_MIGRATIONS` 列表有一条）：**整段初始 schema 都是这一条 migration**，且都用 `STRICT` 表 + `json_valid(record)` 约束。关键索引包括：

```sql
CREATE INDEX entry_heads_by_conversation ON entries (conversation_id, id DESC) WHERE head IS NOT NULL;
CREATE INDEX tasks_by_status ON tasks (status, id);
CREATE INDEX documents_by_address ON documents (kind, scope_kind, owner_id, family, key_value, created_at DESC, retired_at);
```

这些索引对齐三个高频查询：**对话近况**（head entries 倒序）、**活的任务**（scheduler 扫 `pending`/`running`/`waiting`/`completing`）、**当前文档**（按 kind / scope / owner / family / key / created_at / retired_at 定位 incarnation）。

SQLite 后端最大的取舍是 README 那句话："WAL 模式与 `synchronous = NORMAL`：commits 存活于进程崩溃；最新的一笔可能因电源或宿主故障而丢失。" 也就是 SQLite 选择了**不刷盘** —— 数据进 WAL 即算提交，即便 OS 未真正落盘也不损失语义，但遇到 UPS 掉电有微小窗口。

## 4.4 结构与行文：JSONL 后端的 commit 点

`jsonl/storage.ts` 与 SQLite 明显不同：它把记录化整为零，每个 commit 先往 `main.jsonl` 追加 sidecar 记录（预序列化），然后追加一个 commit marker。**marker 的出现是一切原子的前提；marker 之前任何 sidecar 行都是可丢失的候选。**

`FORMAT_VERSION = 1`，同一文件版本只在头部。

pi-durable 对 JSONL 的"commit 点"态度明确：README 上写着 —— "Pass `{ fsync: true }` to flush before each commit marker." 也就是说，没有 fsync 的 JSONL 只保证**提交点在 marker 后**；不做 fsync 时，文件系统可能在 OS 级甚至磁盘级重写部分内容，但语义上（从崩溃到重启读回）"marker 没写成功 = 该 commit 作废"。这就是 `docs/pico-v5-handoff.md` Package 4 的断言："Serialization or preparation failure occurs before file I/O and does not poison the backend."

## 4.5 forked 语义与 schema 版本

`SQLite` 和 `Memory` 后端有一个重要共性：**文档数据有三个 schema 版本字段**。它们分别在 `storedVersion`（存储时不认识的 schema 版本号）、`valueVersion`（当前 tracker 里可访问的 schema 版本号）与 `deltasSinceBase`（自最近一次 base 起 delta 数）。文档创建或迁移时 `valueVersion` 与 `storedVersion` 逐步都会更新到定义的 `version`。

用户可以通过 `defineDoc({ version: 2, migrations: {1: v1ToV2, ...} })` 声明"存储里的版本到运行时版本的迁移"。

## 4.6 三后端一致性自查

后端差异可能带来细微语义差，但接口上全部遵守同一组性质：

1. 同一 Session 单进程内串行（`session/` 层保证）。
2. 提交成功前的所有写不可见，提交成功后的读全部可见。
3. 存储失败 → Session 中毒 → 关闭后开新 Session 才能继续。
4. 文档 schema 版本由三字段 + `migrations` 表达，SQLite/JSONL 不做 SQL 内的 structural 补丁。

用户回答"我能不能用别的后端"有一条明确路径：`testing/` 目录下带 `registerStorageConformance()`（README 举例），任何自定义后端可以套同一份测试套件证明等价性。

## 4.7 阅读清单

- `src/storage/memory.ts:100`——`recordTypes` / `conversations` / `entries` / `tasks` / `submissions` 等 Map。
- `src/storage/sqlite/migrations.ts:5`——initial schema（一个版本号的全部）。
- `src/storage/sqlite/node.ts:12`——`openNodeSqliteStorage()`。
- `src/storage/jsonl/node.ts:14`——`openNodeJsonlStorage()`。
- `src/storage/jsonl/storage.ts:17`——`FORMAT_VERSION` 与 marker 结构。
