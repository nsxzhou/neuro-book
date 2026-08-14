# Directory Structure

> 消费方（`server/`）侧代码如何组织，以及它与本包源码结构的关系。

---

## Overview

本包没有前端目录。消费方代码位于 `server/workspace-files/`，组织方式与包内 `src/`（见 `backend/directory-structure.md`）互补：

- adapter 层：`server/workspace-files/project-file-index.ts`（`ProjectFileIndexAdapter`，组合 `SnapshotCache`）。
- cache identity：`server/workspace-files/workspace-file-index-key.ts`（key 类型 + `projectFileIndexKey` / `plainFileIndexKey` / `workspaceFileIndexKeyId`）。
- 领域类型与 DTO：`server/workspace-files/workspace-files.ts`、`nbook/shared/dto/`（`workspace-tree.dto.ts`、`workspace-file-events.dto.ts`）。
- 测试：`server/workspace-files/project-workspace-index.test.ts`、`workspace-file-index-key.test.ts`。

## Directory Layout

```
server/workspace-files/
├── project-file-index.ts          # ProjectFileIndexAdapter：唯一 SnapshotCache 组合点
├── project-workspace-index.ts     # Project Workspace generation 侧 File Index handle
├── project-workspace-index.test.ts # adapter 行为测试（含 SnapshotClosedError 场景）
├── workspace-file-index-key.ts    # cache identity：kind + scanPolicy 版本化 key
├── workspace-file-index-key.test.ts
├── workspace-files.ts             # WorkspaceFileNode / Issue 领域类型与扫描
├── project-workspace-path-policy.ts
└── runtime-generated-path.ts

packages/file-snapshot-cache/src/   # 包自身结构，见 backend/directory-structure.md
├── index.ts
├── types.ts
├── concurrency.ts
└── snapshot-cache.ts
```

## Module Organization

- **Adapter 只做组合**：`ProjectFileIndexAdapter` 持有唯一 `SnapshotCache` 实例，把 typed target、DTO 与 Project generation handle 组装进 cache 选项；dirty、revision、timer、subscriber、build Promise 等状态**只存在于 cache 内**（见 `project-file-index.ts` 类注释）。
- **identity 单独成文件**：cache key 类型与构造函数集中在 `workspace-file-index-key.ts`；`scanPolicy` 进入 identity，扫描规则升级时不会复用旧 snapshot。
- **包内新增逻辑走 backend 约定**：需要改 cache 内核时改 `packages/file-snapshot-cache/src/`，不要在 `server/` 里复制状态机。

## Naming Conventions

- 消费方文件沿用 kebab-case：`*-index.ts`（adapter）、`*-index-key.ts`（identity）、`*.dto.ts`（DTO，位于 `nbook/shared/dto/`）。
- 包侧公开符号用 `Snapshot` 前缀（`SnapshotCache`、`SnapshotActivation`、`SnapshotClosedError`），消费方不要重命名。
- 错误类按领域命名：包内 `SnapshotClosedError`；消费方自己的错误沿用 `<领域>Error`（如 server 侧 `ProjectFileIndex` 相关错误按 server 约定命名）。

## Examples

- Adapter 组合范例：`server/workspace-files/project-file-index.ts`（构造函数组装 `SnapshotCache`，`readPlain` / `subscribePlain` / `mutatePlain` / `closePlain` 委托 cache）。
- Identity 范例：`server/workspace-files/workspace-file-index-key.ts`（`WorkspaceFileIndexKey` 联合类型 + `scanPolicy` 版本）。
- 消费测试范例：`server/workspace-files/project-workspace-index.test.ts`（覆盖 adapter 的 read/mutate/subscribe/close 与 `SnapshotClosedError` 语义）。
