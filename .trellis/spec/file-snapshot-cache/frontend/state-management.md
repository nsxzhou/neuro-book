# State Management

> 本包无 Pinia/Vuex/前端状态库；「状态」在 cache 的内存 entry 内，消费方**不复制**快照状态。

---

## Overview

- 状态真相源：`SnapshotCache` 私有 `entries: Map<string, SnapshotEntry<...>>`（内存）。每个 entry 的 `generation` / `revision` / `dirty` / `pendingEvents` / `subscribers` 等字段由 cache 独占维护。
- 消费方只通过 `read` / `mutate` / `invalidate` / `subscribe` / `close` 访问；`ProjectFileIndexAdapter` 明确「不保存 dirty、revision、timer、subscriber 或 build Promise」（`server/workspace-files/project-file-index.ts` 类注释）。
- 无持久化：进程重启后状态丢失，由 watcher + builder 重建；「server state」不是从远程拉取，而是消费方自己构建的完整 typed snapshot。

## State Categories

| 类别 | 本包对应现实 |
| --- | --- |
| 本地状态 | cache entry 内部：`snapshot`（最新稳定提交）、`dirty`、`generation`、`revision`、`pendingEvents` / `rawPendingEvents`（有界事件账本）、`timer` / `idleTimer` |
| 全局状态 | `SnapshotCache` 级的 `semaphore`（全局构建并发上限）与 `closed` / `closeAllPromise`；不同 key 的 entry 相互独立 |
| 派生状态 | 消费方把 `FileSnapshot` 转成 DTO（`snapshotDto`），转换逻辑在消费方；`diagnostics()` 是只读派生视图 |
| URL/路由状态 | 不适用 |

## When to Use Global State

- 包内不需要「提升状态」决策：每个 key 的 entry 生命周期由 cache 统一管理，跨 key 只共享全局构建并发（`AsyncSemaphore`，默认 1，消费方如 `ProjectFileIndexAdapter` 设为 2）与全局 closed 标志。
- 消费方侧的「全局」是 cache 实例本身：一个 adapter 持有一个 `SnapshotCache`，需要多个扫描策略时用 identity 区分，而不是开多个 cache 实例。

## Server State

- 读取：`read(key, {staleWhileRevalidate})` 返回最新稳定 snapshot，可选 stale-while-revalidate 后台刷新。
- 变更：`mutate(key, operation)` 与同 key 完整 build 串行，成功/失败后都推进 generation；不同 key 仍可并行。**调用方不得恢复「先写入、再手动 invalidate」两段式协议。**
- 订阅：`subscribe` 只收稳定 commit；raw event 在 rebuild 前单独投递（`onRawEvents`），builder 失败不会吞掉 History 对账机会。
- 回收：无 activation、subscriber、build、debounce 或 pending event 的 entry 默认空闲 5 秒后回收（`idleTtlMs`）；`close(key)` / `closeAll()` 幂等且可重试（watcher close 失败时保留精确 handle 与 closed entry）。

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
/** mutation 成功或失败后都推进 generation：调用方可能已经完成部分写入，cache 不得继续暴露旧 snapshot。 */
async mutate<TResult>(key: TKey, operation: () => TResult | Promise<TResult>): Promise<TResult> {
    const entry = this.ensureEntry(key);
    const task = this.runMutation(entry, operation);
    const settled = task.then(() => undefined, () => undefined);
    entry.mutationPromises.add(settled);
    try {
        return await task;
    } finally {
        entry.mutationPromises.delete(settled);
    }
}
```

## Common Mistakes

- **在消费方缓存 snapshot 副本**：拿到 `FileSnapshot` 后长期持有并假设它是最新；正确做法是再次 `read()` 或订阅 commit。
- **两段式写入**：手动写源数据后再 `invalidate`，与 `mutate` 的串行 + generation 推进冲突，会引入竞态窗口。
- **修改 cache 已接管的数组**：builder 返回 `nodes` / `issues` 后所有权移交 cache，`FileSnapshot` 以 readonly 数组暴露；消费方不得原地修改。
- **不处理 `SnapshotClosedError`**：entry 关闭（idle eviction / `closeAll`）后再 `read` 会 reject；消费方按 `server/workspace-files/project-file-index.ts` 的 `readPlain` 模式捕获并重试。
