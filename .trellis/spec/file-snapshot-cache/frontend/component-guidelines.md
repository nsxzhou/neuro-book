# Component Guidelines

> 本包无前端组件；「组件」的等价物是消费方的 **Adapter 组合**与 **Handle** 结构。

---

## Overview

本包不提供 Vue/React 组件，不适用 props/composition/a11y 约定。等价现实：

- 消费方「组件」= 领域 Adapter（`ProjectFileIndexAdapter`），构造函数一次性组装 `SnapshotCache` 选项，公开方法委托 cache。
- 「props」= `SnapshotCacheOptions` 泛型契约与 Adapter 自己的只读选项类型（`ProjectFileIndexAdapterOptions`）。
- 「composition」= Adapter 把 typed target、builder、watcher、DTO 组装进 cache，自己不持有快照状态。

## Component Structure

Adapter 的标准结构（真实例子 `server/workspace-files/project-file-index.ts`）：

1. `constructor(options)` 创建唯一 `SnapshotCache`，绑定 `keyId` / `builder` / `watcher` / `eventId` / `debounceMs` / `maxConcurrentBuilds`。
2. 公开方法（`readPlain` / `subscribePlain` / `waitPlainReady` / `mutatePlain` / `closePlain`）委托 cache 并做领域转换（`snapshotDto`）。
3. 私有辅助（`closePlainLease` / `finishPlainClose`）管理引用计数 lease 与重试。

```typescript
// server/workspace-files/project-file-index.ts
this.cache = new SnapshotCache({
    keyId: (key) => workspaceFileIndexKeyId(key.identity),
    builder: {
        build: ({key, signal}) => options.build({
            target: key.target,
            workspace: key.workspace,
            signal,
        }),
    },
    watcher: {
        open: ({key, signal, onEvent, onError}) => options.openWatcher({
            target: key.target,
            workspace: key.workspace,
            signal,
            onEvent,
            onError,
        }),
    },
    eventId: (event) => event.path,
    debounceMs: FILE_INDEX_REBUILD_DEBOUNCE_MS,
    maxConcurrentBuilds: 2,
});
```

## Props Conventions

- 包侧配置全部可选并带默认值（`debounceMs` 默认 120、`maxPendingEvents` 默认 1_000、`maxSubscribers` 默认 1_000、`maxConcurrentBuilds` 默认 1、`maxBuildAttempts` 默认 3、`idleTtlMs` 默认 5_000）；配置校验失败抛普通 `Error`（`assertPositiveInteger` / `assertNonNegativeInteger`）。
- 类型参数按 `TKey, TNode, TIssue, TEvent` 顺序固定；消费方显式传入完整领域类型（如 `SnapshotCache<FileIndexCacheKey, WorkspaceFileNode, WorkspaceFileIssue, WorkspaceFileChangeEventDto>`），不用 `any` 消音。
- 公开结构字段尽量 `readonly`（`SnapshotActivation.ready` / `close()`、`ProjectFileIndexAdapterOptions` 各字段只读）。

## Styling Patterns

不适用：本包与消费侧无样式层。`AGENTS.md` 中主题变量、Tailwind、`dark:` 变体等约定只作用于 `web/`，与本包无关。

## Accessibility

不适用：无 UI 渲染。等价要求是**资源可达性**——长期消费者必须显式持有 `activate()` 返回的 handle、等待 `ready`、并在结束时 `close()`，否则 entry 只能靠 idle eviction 回收。

## Common Mistakes

- **在 Adapter 里复制 cache 状态机**：保存 dirty/revision/timer 副本会与 cache 内真相分叉（`project-file-index.ts` 类注释明确「不保存 dirty、revision、timer、subscriber 或 build Promise」）。
- **泄漏 activation / 订阅**：`subscribePlain` 用引用计数 lease（`PlainActivationLease`），首个消费者 activate、最后一个消费者 close；不递减引用会让 watcher 常驻。
- **把领域逻辑塞进包内**：扫描、ignore 规则、DTO 归一化必须留在消费方 adapter / builder / watcher，包内禁止出现 NeuroBook 标识（`tests/isolation.test.ts` 强制）。
