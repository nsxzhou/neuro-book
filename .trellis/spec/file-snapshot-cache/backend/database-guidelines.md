# Database Guidelines

> 本包**没有数据库层**：`@notnotype/file-snapshot-cache` 是纯内存快照缓存。以下各节按模板结构如实说明「无数据库」时的对应现实。

---

## Overview

本包不连接 SQLite、libsql、Prisma 或任何持久化存储，也不提供磁盘 projection/store。完整 snapshot 只驻留内存；进程重启即丢失，由调用方重新构建。这条边界被 `tests/isolation.test.ts` 强制检查（源码不得出现 `sqlite|libsql|prisma`、`.nbook`、`frontmatter` 等，且不得存在 `json-projection-store.ts` 之类无生产消费者的接口）。

「数据」实际存储方式是 `SnapshotCache` 私有的内存 `Map`：

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
private readonly entries = new Map<string, SnapshotEntry<TKey, TNode, TIssue, TEvent>>();
```

每个 entry 持有最新稳定 `FileSnapshot`（`nodes` / `issues` 只读数组）与世代计数：

```typescript
// packages/file-snapshot-cache/src/types.ts
export interface FileSnapshot<TNode, TIssue> {
    readonly nodes: readonly TNode[];
    readonly issues: readonly TIssue[];
    readonly revision: number;
    readonly generation: number;
    readonly calculatedAt: string;
}
```

## Query Patterns

本包没有 SQL 查询。对内存数据的访问统一通过 `SnapshotCache` 的公开方法，调用方不得直接触碰 entry 内部：

- `read(key, {staleWhileRevalidate})`：读取最新稳定 snapshot；同 key 并发共享一个 build。
- `mutate(key, operation)`：与同 key 的 build 串行执行 mutation，成功或失败后都推进 generation。
- `invalidate(key, event)`：标记外部变化，事件按 `eventId` 有界归并，debounce 后后台重建。

「查询」的等价物是 key 到 entry 的映射：`keyId(key)` 决定 entry 身份，相同的 id 复用同一 entry（`ensureEntry` 先查 `this.entries` / `this.closePromises`）。真实例子：

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
async read(key: TKey, options: SnapshotReadOptions = {}): Promise<FileSnapshot<TNode, TIssue>> {
    const entry = this.ensureEntry(key);
    try {
        if (entry.snapshot && !entry.dirty) {
            return entry.snapshot;
        }
        if (entry.snapshot && options.staleWhileRevalidate) {
            this.startBackgroundBuild(entry);
            return entry.snapshot;
        }
        return await this.stabilize(entry);
    } finally {
        this.scheduleIdle(entry);
    }
}
```

## Migrations

无数据库，因此没有 schema migration。世代管理由内存计数器承担：

- `generation`：外部变更/失败 mutation 的累计次数（`markDirty` 时 `entry.generation += 1`）。
- `revision`：稳定提交次数（每次成功 commit 时 `entry.revision + 1`）。
- 构建期间 generation 变化即丢弃旧结果重试（`buildUntilStable` 中 `if (generation !== entry.generation) { entry.discardedBuildCount += 1; continue; }`），永不提交已知过期 snapshot。

跨进程/重启的「版本迁移」等价物在消费方：cache identity 里携带 schema/扫描规则版本，规则升级时不会复用旧 snapshot。真实例子：

```typescript
// server/workspace-files/workspace-file-index-key.ts
export type WorkspaceFileIndexKey =
    | Readonly<{
        kind: "project";
        projectKey: ProjectWorkspaceKey;
        scanPolicy: "project-v1";
    }>
    | Readonly<{
        kind: "plain-workspace";
        root: AbsoluteFsPath;
        scanPolicy: "plain-v1";
    }>;
```

## Naming Conventions

无表名/列名/索引名。对应现实：

- entry 身份：`id`（`keyId` 的返回值，字符串）。
- 世代字段：`generation`（变更轮次）、`revision`（稳定提交轮次）、`dirty`（是否有未提交变更）、`building`（是否在构建）。
- 有界事件账本：`pendingEvents` / `rawPendingEvents`（Map，按 `eventId` 去重）、`droppedEventCount` / `rawDroppedEventCount`。

## Common Mistakes

- **给本包加磁盘 projection/store**：无生产消费者，会被 `tests/isolation.test.ts` 拒绝（断言源码不含 `JsonProjectionStore` / `PersistedProjection` / `readProjection` 等标识，且 `package.json` 不导出 `./node`）。需要持久化的场景由消费方在自己的存储层实现。
- **假设进程重启后 snapshot 还在**：本包内存态不持久，长期 owner 必须显式 `activate` 并等待 `ready`，用 watcher + builder 重建。
- **绕过公开方法直接改 entry**：`SnapshotEntry` 是包内私有类型（未导出），消费方只能通过 `read` / `mutate` / `invalidate` / `subscribe` / `close` 访问。
