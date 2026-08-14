# Directory Structure

> `packages/file-snapshot-cache/` 的源码组织方式。

---

## Overview

本包是单职责的纯 TS 库，`src/` 采用扁平结构，不按功能再分子目录：

- 所有公开类型与错误类集中在 `src/types.ts`（接口 + 两个 Error 子类）。
- 通用并发原语 `AsyncSemaphore` 独立成 `src/concurrency.ts`。
- 核心业务逻辑是 `src/snapshot-cache.ts` 中的 `SnapshotCache` class（高领域逻辑，遵循 `AGENTS.md`「后端高领域逻辑使用 class」）。
- `src/index.ts` 只做 barrel 重导出，不包含实现。
- 本包没有 API 端点、没有路由、没有 ORM；「对外入口」就是 `src/index.ts` 重导出的 `SnapshotCache` 与类型。

包内不使用相对路径导入，一律走 `package.json` `imports` / `tsconfig.json` `paths` 定义的 `#cache/*`（源码）与 `#test/*`（测试）别名：

```typescript
// packages/file-snapshot-cache/src/snapshot-cache.ts
import {AsyncSemaphore} from "#cache/concurrency";
import {
    SnapshotClosedError,
    SnapshotUnstableError,
    type FileSnapshot,
    type SnapshotActivation,
    // ...
} from "#cache/types";
```

---

## Directory Layout

```
packages/file-snapshot-cache/
├── package.json          # exports "." -> ./src/index.ts；imports 定义 #cache/* 与 #test/*
├── tsconfig.json         # strict、noEmit、moduleResolution Bundler、paths 映射 #cache/*
├── vitest.config.ts      # node 环境、include tests/**/*.test.ts、testTimeout 20_000
├── README.md             # 公共 API 与 Boundary 说明（中文）
├── src/
│   ├── index.ts          # barrel：重导出 SnapshotCache、类型与错误类
│   ├── types.ts          # 全部公开 interface + SnapshotClosedError / SnapshotUnstableError
│   ├── concurrency.ts    # AsyncSemaphore（全局构建并发上限用）
│   └── snapshot-cache.ts # SnapshotCache class：entry 生命周期、generation、build/mutate/close
├── tests/
│   ├── helpers.ts        # 测试夹具：cacheOptions、buildResult、deferred、waitFor
│   ├── concurrency.test.ts
│   ├── snapshot-cache.test.ts
│   └── isolation.test.ts # 包边界自检：禁止依赖 nbook/nuxt/sqlite 等领域
└── benchmarks/
    ├── run.ts            # Node/Bun 双运行基准，写入 benchmarks/results/
    └── results/          # baseline-node.md / baseline-bun.md（本机生成物，不提交）
```

---

## Module Organization

- 新增一个可复用原语（如信号量、重试器）时，像 `concurrency.ts` 一样独立成文件，并让 `snapshot-cache.ts` 用 `#cache/*` 别名导入。
- 新增公开类型/错误类放 `src/types.ts`；新增公开实现入口在 `src/index.ts` 用 `export {X} from "#cache/..."` / `export type {...} from "#cache/types"` 重导出。
- 不要在 `src/index.ts` 里写业务逻辑；它是纯 re-export 桶（真实例子见 `src/index.ts`）。

```typescript
// packages/file-snapshot-cache/src/index.ts
export {SnapshotCache} from "#cache/snapshot-cache";
export type {
    FileSnapshot,
    SnapshotBuildResult,
    SnapshotBuilder,
    // ...
} from "#cache/types";
export {SnapshotClosedError, SnapshotUnstableError} from "#cache/types";
```

- 领域适配不放本包：`server/workspace-files/project-file-index.ts` 的 `ProjectFileIndexAdapter` 才是 NeuroBook 领域侧组合层（见 `frontend/` 目录）。

## Naming Conventions

- 文件/目录：kebab-case（`snapshot-cache.ts`、`concurrency.ts`、`project-file-index.ts`）；测试文件以 `.test.ts` 结尾，基准为 `benchmarks/run.ts`。
- class：PascalCase（`SnapshotCache`、`AsyncSemaphore`）。
- 公开接口：以 `Snapshot` / `FileSnapshot` 前缀标识与快照领域的关系（`SnapshotBuildResult`、`SnapshotActivation`、`SnapshotCommit`、`SnapshotWatcher`、`SnapshotEntryDiagnostics`）。
- 错误类：`<领域>Error` 后缀（`SnapshotClosedError`、`SnapshotUnstableError`），放在 `types.ts` 并在 `index.ts` 重导出。
- 类型参数：固定 `TKey, TNode, TIssue, TEvent`（见 `SnapshotCache<TKey, TNode, TIssue, TEvent>`），全包一致。

## Examples

- 组织良好的核心模块：`packages/file-snapshot-cache/src/snapshot-cache.ts`（入口类）、`src/types.ts`（契约集中）、`src/concurrency.ts`（单一原语）。
- 测试夹具组织范例：`packages/file-snapshot-cache/tests/helpers.ts`（`cacheOptions` 返回默认配置，`deferred` / `waitFor` 精确控制竞态边界）。
- 桶文件范例：`packages/file-snapshot-cache/src/index.ts`。
