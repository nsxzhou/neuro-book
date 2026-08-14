# Hook Guidelines

> 本包无 React hooks / Vue composables；「hook」的等价物是**生命周期句柄**与**订阅回调 Seam**。

---

## Overview

本包暴露的「可挂接点」全部是句柄与回调，不是前端 hooks：

- `SnapshotActivation`：`activate(key)` 同步返回 `{ready, close}`，`ready` 表示 watcher 打开成功，`close` 释放该 activation。
- `subscribe(key, subscriber)`：返回幂等取消函数；只接收成功的稳定 commit。
- `SnapshotWatcher.open(...)`：接收 `{key, signal, onEvent, onError}`，返回可确定性关闭、可安全重试的 `SnapshotWatchHandle`。
- `onRawEvents`：activation 首次建立时原子绑定，每次 rebuild 前接收未投递的 raw event batch。

## Custom Hook Patterns

生命周期句柄模式（真实例子 `packages/file-snapshot-cache/src/snapshot-cache.ts`）：

```typescript
activate(key: TKey, options: SnapshotActivationOptions<TEvent> = {}): SnapshotActivation {
    const entry = this.ensureEntry(key);
    if (entry.activation) {
        return entry.activation;
    }
    entry.onRawEvents = options.onRawEvents ?? null;
    const activation: SnapshotActivation = {
        ready: this.openWatcher(entry),
        close: () => this.closeExactEntry(entry),
    };
    entry.activation = activation;
    return activation;
}
```

订阅返回幂等取消函数（多次调用无副作用，并触发 `scheduleIdle` 以便空闲回收）：

```typescript
subscribe(key: TKey, subscriber: Subscriber<TNode, TIssue, TEvent>): () => void {
    const entry = this.ensureEntry(key);
    if (entry.subscribers.size >= this.maxSubscribers) {
        throw new Error(`snapshot ${entry.id} exceeded maxSubscribers=${this.maxSubscribers}`);
    }
    entry.subscribers.add(subscriber);
    let subscribed = true;
    return () => {
        if (!subscribed) {
            return;
        }
        subscribed = false;
        entry.subscribers.delete(subscriber);
        this.scheduleIdle(entry);
    };
}
```

消费方按引用计数 lease 组合 activation 与订阅（`server/workspace-files/project-file-index.ts` 的 `subscribePlain`：首个消费者 `activate`，最后一个消费者 `closePlainLease`）。

## Data Fetching

本包没有数据获取框架（非 React Query / SWR）。等价物：

- `read(key, {staleWhileRevalidate: true})`：立即返回 stale snapshot 并后台刷新（`startBackgroundBuild`）。
- `read(key)`：等待最新稳定 snapshot；同 key 并发 reader 共享一个 build，失败后下一批 reader 共享一个重试 Promise。
- `subscribe(key, fn)`：接收稳定 commit（含 `events` / `droppedEventCount`）。
- watcher 不由 `read()` / `subscribe()` 隐式打开；长期 owner 必须显式 `activate` 并 `await activation.ready`。

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
if (entry.snapshot && options.staleWhileRevalidate) {
    this.startBackgroundBuild(entry);
    return entry.snapshot;
}
```

## Naming Conventions

- `use*` 前缀不适用（无 hooks）。包内固定动词：`activate` / `read` / `mutate` / `invalidate` / `subscribe` / `close` / `closeAll`。
- 句柄与回调类型用 `Snapshot` 前缀：`SnapshotActivation`、`SnapshotWatchHandle`、`SnapshotWatcher`、`SnapshotRawEventBatch`。
- 取消函数用「返回函数」而非 `unsubscribe` 对象字段（`subscribe` 返回 `() => void`，幂等）。

## Common Mistakes

- **忘记 close activation**：`activate` 后不 `close()`，entry 永不空闲回收、watcher 常驻；消费方必须在 finally / lease 归零路径关闭。
- **把 `subscribe` 当隐式 watcher**：订阅只收稳定 commit，不打开 watcher；需要实时性必须 `activate` + `ready`。
- **不处理取消函数的幂等性**：多次调用取消函数应有副作用（包内实现已保证），消费方不要在外部重复维护 `subscribed` 标志。
- **在回调里抛错**：subscriber / `onRawEvents` 的同步或异步错误都会被 cache 吞掉（不污染 commit），消费方应自行捕获并在日志/诊断中记录。
