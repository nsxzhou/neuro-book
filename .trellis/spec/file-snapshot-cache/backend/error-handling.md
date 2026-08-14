# Error Handling

> 本包的错误处理约定：typed 错误类 + promise rejection + 有界 diagnostics。

---

## Overview

- 错误类型：两个公开错误类 `SnapshotClosedError`、`SnapshotUnstableError`，定义在 `src/types.ts`；配置校验失败抛普通 `Error`（消息形如 `"debounceMs must be a non-negative integer"`）。
- 传播方式：异步错误一律以 rejected promise 向调用方传播（`read` / `mutate` / `close` / `closeAll` 都是 async）；同步校验直接 `throw`。
- 记录方式：本包不写日志；build 失败会被截断投影进 `diagnostics()`（`lastBuildError` / `lastBuildFailedAt` / `buildFailureCount`）。
- 返回给调用方：无 HTTP API；错误通过 `instanceof SnapshotClosedError` 等 typed 判定消费（见 `server/workspace-files/project-file-index.ts`）。

## Error Types

```typescript
// packages/file-snapshot-cache/src/types.ts
/** cache 已关闭或 key 在构建期间关闭。 */
export class SnapshotClosedError extends Error {
    constructor(message = "snapshot cache entry is closed") {
        super(message);
        this.name = "SnapshotClosedError";
    }
}

/** 连续变更使 snapshot 无法在有限尝试内稳定。 */
export class SnapshotUnstableError extends Error {
    constructor(readonly keyId: string, readonly attempts: number) {
        super(`snapshot ${keyId} did not stabilize after ${attempts} build attempts`);
        this.name = "SnapshotUnstableError";
    }
}
```

两个错误类都在 `src/index.ts` 重导出，消费方从包入口导入后即可用 `instanceof` 判定。

## Error Handling Patterns

- **AbortSignal 传递**：`AbortController` 贯穿 watcher、build、semaphore；abort 时把 `signal.reason` 原样抛出，不吞掉原因：

```typescript
// packages/file-snapshot-cache/src/concurrency.ts
async acquire(signal: AbortSignal): Promise<() => void> {
    if (signal.aborted) {
        throw signal.reason;
    }
    // ...
}
```

- **每个异步边界后复查 entry 状态**（`assertEntryOpen`）：closed 或 aborted 时抛 `signal.reason`，隔离 late result：

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
function assertEntryOpen<TKey, TNode, TIssue, TEvent>(
    entry: SnapshotEntry<TKey, TNode, TIssue, TEvent>,
    signal: AbortSignal,
): void {
    if (entry.closed || signal.aborted) {
        throw signal.reason instanceof Error ? signal.reason : new SnapshotClosedError(`snapshot ${entry.id} is closed`);
    }
}
```

- **mutation 失败仍推进 generation**：`runMutation` 的 `finally` 里只要 generation 未变就 `markDirty`，调用方可能已部分写入，cache 不得继续暴露旧 snapshot。
- **边界回调错误相互隔离**：subscriber / raw event callback 的同步与异步错误都被吞掉，不污染 commit 与其他订阅者（`notifySubscribers` / `notifyRawEvents` 中 `void Promise.resolve(...).catch(() => undefined)` 包 `try/catch`）。
- **build 失败进有界 diagnostics 而非崩溃**：`boundedDiagnosticError` 把任意异常投影为单条定长文本（最长 2_000 字符，超出截断加 `...`），同时 `buildFailureCount += 1`；下一批 reader 只共享一个重试 Promise。
- **消费方重试模式**：`server/workspace-files/project-file-index.ts` 的 `readPlain` 捕获 `SnapshotClosedError` 后完成 pending close 再重读一次：

```typescript
try {
    return snapshotDto(await this.cache.read(key));
} catch (error) {
    if (!(error instanceof SnapshotClosedError)) {
        throw error;
    }
    const id = workspaceFileIndexKeyId(key.identity);
    await this.finishPlainClose(id, error);
    return snapshotDto(await this.cache.read(key));
}
```

## API Error Responses

本包是库，没有 HTTP API，因此没有标准错误响应格式。等价现实：

- 同步错误：抛普通 `Error`（配置校验），见 `assertPositiveInteger` / `assertNonNegativeInteger`。
- 异步错误：rejected promise，错误对象尽量保持 typed（`SnapshotClosedError` / `SnapshotUnstableError` / 原始 `signal.reason`）。
- 可观测面：`diagnostics()` 返回的 `SnapshotEntryDiagnostics.lastBuildError`（有界字符串）与 `watcherError`。

## Common Mistakes

- **新增错误类忘记在 `index.ts` 重导出**：调用方只能 `instanceof` 包入口导出的类；未导出的类会让 typed 判定失效。
- **吞掉 build 错误却不记录 diagnostics**：`stabilize` 的 catch 必须走 `boundedDiagnosticError` + 计数，否则失败不可观测、重试语义丢失。
- **await 之后不复查 entry 状态**：watcher open、build 返回后必须 `assertEntryOpen`，否则 late result 会被提交到已关闭/已重开的 entry（`closeExactEntry` 保证按 incarnation 关闭，但调用方仍需在边界复查）。
- **抛非 Error 值**：内部统一 `throw signal.reason`（通常是 Error）或 `new Error(...)`；`boundedDiagnosticError` 对非 Error 值用 `String(error)` 兜底，调用方不要依赖 catch 到字符串。
