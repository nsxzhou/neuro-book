# Quality Guidelines

> 消费方（`server/`）侧的质量标准：与包侧同源的类型/测试纪律，外加 Adapter 层的正确性要求。

---

## Overview

- 消费方改动同样受仓库质量门禁约束：`bun run typecheck`（根）与相关 focused vitest 测试（`bun run test -- path/to/relevant.test.ts`）；`server/` 侧测试覆盖 adapter 行为（如 `server/workspace-files/project-workspace-index.test.ts`）。
- 消费方遵循 `AGENTS.md`：`server/` 使用 `nbook/...` 绝对路径别名（如 `nbook/server/workspace-files/project-workspace`），不使用相对路径导入；后端高领域逻辑用 class（`ProjectFileIndexAdapter`）。
- 本包侧质量标准见 `backend/quality-guidelines.md`，消费方不得用 cast 绕过包的类型边界。

## Forbidden Patterns

- **在 `server/` 复制 cache 状态机**：adapter 不保存 dirty/revision/timer/build Promise（`project-file-index.ts` 类注释明示）；违反会造成状态分叉。
- **把领域逻辑塞进包内**：包 `src/` 不得出现 `nbook/`、nuxt、sqlite 等领域标识（`tests/isolation.test.ts` 强制）；扫描、ignore、DTO 归一化留在消费方。
- **相对路径导入**：`server/` 用 `nbook/...` 别名；包内用 `#cache/*` / `#test/*`；不要新引入 `../` 链。
- **两段式写入**：调用方不得「先写入、再手动 invalidate」；mutation 必须走 `mutate()`（或按 lease 语义调用 adapter 的 `mutatePlain`）。
- **`any` / untyped cast**：adapter 构造 `SnapshotCache` 时显式填四个泛型参数，不 `as any`。

## Required Patterns

- Adapter 构造函数一次性组装 cache 选项（`keyId` / `builder` / `watcher` / `eventId` / `debounceMs` / `maxConcurrentBuilds`），真实例子见 `server/workspace-files/project-file-index.ts`：

```typescript
this.cache = new SnapshotCache({
    keyId: (key) => workspaceFileIndexKeyId(key.identity),
    builder: {
        build: ({key, signal}) => options.build({
            target: key.target,
            workspace: key.workspace,
            signal,
        }),
    },
    // ...
    eventId: (event) => event.path,
    debounceMs: FILE_INDEX_REBUILD_DEBOUNCE_MS,
    maxConcurrentBuilds: 2,
});
```

- 处理 `SnapshotClosedError` 重试：`readPlain` 捕获后完成 pending close 再重读（见 `frontend/type-safety.md` / `backend/error-handling.md` 的代码片段）。
- lease 引用计数：`subscribePlain` 首个消费者 activate、最后一个消费者 close；`closePlain` 在后台 close 失败时重试同一 activation handle。
- 测试断言不重放副作用：mutation callback 抛 `SnapshotClosedError` 时 `toHaveBeenCalledOnce()`（`project-workspace-index.test.ts`）。

## Testing Requirements

- 消费侧行为用 vitest 覆盖 adapter 而非直接测包内部（包内部测试在 `packages/file-snapshot-cache/tests/`）。
- 竞态/异步用 `vi.waitFor`（`project-workspace-index.test.ts` 中 `await vi.waitFor(() => expect(events).toContain("workspace_watch_ready"))`）与 `vi.fn` mock builder/watcher。
- 每次用例清理资源（`closePlain` / `cache.closeAll`），避免 watcher/timer 泄漏。
- 报告验证时区分 focused test / typecheck / 浏览器验收，不能互相替代（`PROJECT-STATUS.md` 验证口径）。

## Code Review Checklist

- [ ] Adapter 不保存 cache 内部状态副本；所有生命周期委托 `SnapshotCache`。
- [ ] `import type` 用于纯类型；泛型 `TKey/TNode/TIssue/TEvent` 显式填写，无 `any`。
- [ ] `SnapshotClosedError` 路径有捕获与重试；mutation 走 `mutate()`，无两段式写入。
- [ ] activation/subscription 成对出现（lease 引用计数正确，最后一个消费者 close）。
- [ ] 包 `src/` 保持领域无关（`isolation.test.ts` 绿色），新增逻辑落在正确的层。
- [ ] 相关 focused test 通过且清理资源；报告注明未运行的检查。
