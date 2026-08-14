# Type Safety

> 本包的消费侧类型安全约定：泛型完整类型化、错误类 `instanceof` 判定、无运行时校验库。

---

## Overview

- 类型系统：TypeScript `strict: true`（`tsconfig.json`），本包与消费侧都保持类型完整。
- 类型组织：包内所有公开类型集中在 `src/types.ts`，经 `src/index.ts` 用 `export type {...}` 重导出；消费方从包入口 `import type`。
- 校验库：本包**不使用 Zod / Yup / io-ts 等运行时校验库**；配置用 `assertPositiveInteger` / `assertNonNegativeInteger` 手写校验，错误用 typed Error 类表达。
- 类型推断：公开 API 尽量让推断自然成立（如 `read` 返回 `Promise<FileSnapshot<TNode, TIssue>>`），测试断言用 `satisfies` 保持类型约束。

## Type Organization

- 包内：接口与错误类都在 `src/types.ts`；`SnapshotCache<TKey, TNode, TIssue, TEvent>` 四个泛型参数全包一致；`src/index.ts` 是唯一对外类型入口。
- 消费方：领域类型在 `server/workspace-files/`（`WorkspaceFileNode`、`WorkspaceFileIssue`、`WorkspaceFileChangeEventDto` 等），DTO 在 `nbook/shared/dto/`；adapter 把领域类型显式填入泛型：

```typescript
// server/workspace-files/project-file-index.ts
private readonly cache: SnapshotCache<FileIndexCacheKey, WorkspaceFileNode, WorkspaceFileIssue, WorkspaceFileChangeEventDto>;
```

- 测试侧：`tests/helpers.ts` 通过 `import type {SnapshotBuildResult, SnapshotCacheOptions} from "#cache/index"` 引用包类型，测试夹具类型与生产契约同源。

## Validation

- 无 schema 校验；配置校验在 `SnapshotCache` 构造函数内同步执行，失败抛普通 `Error`：

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
function assertPositiveInteger(value: number, name: string): number {
    if (!Number.isInteger(value) || value <= 0) {
        throw new Error(`${name} must be a positive integer`);
    }
    return value;
}
```

- 运行期「类型判定」用 `instanceof`（`SnapshotClosedError`）或 `toMatchObject` + `satisfies`（测试断言 `SnapshotUnstableError` 字段）：

```typescript
// packages/file-snapshot-cache/tests/snapshot-cache.test.ts
await expect(cache.read("alpha")).rejects.toMatchObject({
    name: "SnapshotUnstableError",
    attempts: 2,
} satisfies Partial<SnapshotUnstableError>);
```

- 包边界由 `tests/isolation.test.ts` 以源码文本断言强制（禁止领域依赖、禁止 production dependency、禁止死接口），属于「契约测试」而非运行时校验。

## Common Patterns

- `readonly` 表达不可变契约：`FileSnapshot.nodes: readonly TNode[]`、`SnapshotCommit.events: readonly TEvent[]`；消费方不得修改 cache 已接管的数组。
- `import type` 分离：纯类型导入一律 `import type {...} from "#cache/types"`（包内）或 `import type {...} from "@notnotype/file-snapshot-cache"`（消费方），值导入只用于 `SnapshotCache`、`SnapshotClosedError`、`SnapshotUnstableError`。
- 泛型贯穿：`SnapshotBuilder` / `SnapshotWatcher` / `SnapshotActivation` 全部以 `TKey, TNode, TIssue, TEvent` 参数化，避免在消费方做 untyped cast。
- 错误类带结构化字段：`SnapshotUnstableError` 携带 `keyId` 与 `attempts`（`readonly` 构造参数），消费方无需 parse message。

## Forbidden Patterns

- **`any` 或 `as unknown as T` 消音**：`src/` 无任何 `any`；消费方遇到类型缺口应修正泛型或类型定义，不要 cast 绕过（`.trellis/spec/guides/cross-layer-thinking-guide.md` 同样禁止 UI/命令代码直接 cast 未类型化载荷）。
- **手写重复类型替代包类型**：消费方若复制 `FileSnapshot` 结构会与包合同分叉；一律从 `@notnotype/file-snapshot-cache` 导入。
- **依赖 `any`-化错误处理**：`catch (error)` 后不做 `instanceof` 判定就直接用，会丢失 typed 错误语义（如 `SnapshotClosedError` 重试逻辑）。
- **在包内引入运行时校验框架**：与「零生产依赖 + 无数据库/无框架」的边界冲突（`tests/isolation.test.ts` 会拒绝新依赖）。
