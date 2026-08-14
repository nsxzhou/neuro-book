# Logging Guidelines

> 本包**不写日志**：`src/` 中没有 logger 也没有 `console.*` 输出。日志约定主要适用于消费方（`server/` 侧）。

---

## Overview

- 本包作为库不产生日志，避免向宿主进程注入日志实现；`AGENTS.md` 的「结构化日志：`this.logger.debug({ kind: message.kind }, "...")`」约定适用于 `server/` 消费方，不适用于本包。
- 本包的可观测机制是 `diagnostics()`：返回有界、结构化的资源状态，供消费方/测试断言，不依赖日志框架。

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
diagnostics(): SnapshotCacheDiagnostics {
    const entries: Record<string, SnapshotEntryDiagnostics> = {};
    // ...
    return {
        entryCount: this.entries.size,
        activeBuildCount: this.semaphore.activeCount,
        queuedBuildCount: this.semaphore.queuedCount,
        timerCount,
        idleTimerCount,
        watcherCount,
        watcherOpeningCount,
        subscriberCount,
        entries,
    };
}
```

## Log Levels

本包无日志级别。等价现实是 diagnostics 的「程度」字段：

- `buildFailureCount` / `lastBuildError` / `lastBuildFailedAt`：对应 error 级信息（构建失败）。
- `watcherError`：对应 warn/error 级信息（watcher 打开或运行失败，打开失败只标记诊断，read-time build 仍可工作）。
- `stableCommitCount` / `discardedBuildCount` / `droppedEventCount`：对应 info/debug 级信息（提交、丢弃、事件丢失）。

消费方需要在日志里区分级别时，按 `AGENTS.md` 约定在 `server/` 侧记录（如 build 失败用 error、watcher ready 用 info）。

## Structured Logging

- 本包侧：`SnapshotEntryDiagnostics` 本身就是结构化字段集合（`generation`、`revision`、`dirty`、`building`、`pendingEventCount`、`droppedEventCount`、`subscriberCount`、`buildCount` 等），消费方与测试用 `toMatchObject` 断言这些字段：

```typescript
// packages/file-snapshot-cache/tests/snapshot-cache.test.ts
expect(cache.diagnostics().entries.alpha).toMatchObject({
    dirty: true,
    building: false,
    buildCount: 1,
    buildFailureCount: 1,
    lastBuildFailedAt: "2026-07-24T12:00:00.000Z",
});
```

- 消费方侧：`server/` 使用结构化日志（`this.logger.debug({ kind: message.kind }, "...")`），不要把快照内容整包打进日志。
- `boundedDiagnosticError` 保证错误文本有界（最长 2_000 字符，截断加 `...`），防止大错误对象膨胀诊断：

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
function boundedDiagnosticError(error: unknown): string {
    let message = "Unknown build failure";
    try {
        message = error instanceof Error
            ? `${error.name}: ${error.message}`
            : String(error);
    } catch {
        // 异常对象的自定义字符串转换不能覆盖真正的builder failure。
    }
    return message.length <= MAX_DIAGNOSTIC_ERROR_LENGTH
        ? message
        : `${message.slice(0, MAX_DIAGNOSTIC_ERROR_LENGTH - 3)}...`;
}
```

## What to Log

本包自身不产生日志事件。值得消费方在 `server/` 侧记录的事件（按需）：

- build 失败：`buildFailureCount` 增长、`lastBuildError` 有值。
- watcher 打开失败 / 运行错误：`watcherError` 有值。
- 资源生命周期：`activate` / `subscribe` / `close` / idle eviction（`diagnostics().entryCount`、`watcherCount` 变化）。

## What NOT to Log

- **快照内容（`nodes` / `issues`）**：可能是完整文件树，整包进日志会膨胀且泄露内容；只记录计数与世代。
- **PII / 密钥 / 原始事件载荷**：消费方 adapter 决定什么进日志，`AGENTS.md` 与 `CONTRIBUTING.md` 明确不提交 `.env`、API Key、Session、Trace、日志等。
- **未截断的错误对象**：build 错误必须经 `boundedDiagnosticError`（2_000 字符上限），不要把完整 Error/cause 对象图写入诊断。
