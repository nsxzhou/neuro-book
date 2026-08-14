# Frontend Development Guidelines

> `@notnotype/file-snapshot-cache` 的「前端」层约定——本包**没有前端**，本目录记录的是被 `server/` 消费的契约与类型安全约定。

---

## Overview

本包是纯 TS 内存快照缓存库，不包含 Vue 组件、React hooks 或浏览器代码。它的消费者是 `server/`（Nuxt/Nitro 服务端），例如 `server/workspace-files/project-file-index.ts` 的 `ProjectFileIndexAdapter` 与 `server/workspace-files/project-workspace-index.test.ts`。

因此 `frontend/` 目录不记录组件/样式约定，而是记录**消费契约**：

- 如何组合 `SnapshotCache`（builder / watcher / keyId / eventId 等选项）。
- 快照生命周期（`activate` → `ready` → `read` / `mutate` / `subscribe` → `close`）。
- 类型安全与错误约定（`SnapshotClosedError`、泛型完整类型化、`readonly` 契约）。

`AGENTS.md` 中关于 `web/` 前端（Composition API、主题变量、`useNotification` 等）的规则与本包无关，不适用。

真实消费方式（`server/workspace-files/project-file-index.ts`）：

```typescript
import {
    SnapshotCache,
    SnapshotClosedError,
    type SnapshotActivation,
} from "@notnotype/file-snapshot-cache";
```


---

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | 消费方文件组织：adapter、identity key、DTO | Filled |
| [Component Guidelines](./component-guidelines.md) | 无组件；记录 Adapter 组合与 Handle 结构 | Filled |
| [Hook Guidelines](./hook-guidelines.md) | 无 hooks；记录生命周期句柄与订阅回调 Seam | Filled |
| [State Management](./state-management.md) | 无前端状态库；状态在 cache entry 内，消费方不复制 | Filled |
| [Quality Guidelines](./quality-guidelines.md) | 消费方侧质量标准：类型完整、错误重试、不复制状态机 | Filled |
| [Type Safety](./type-safety.md) | 泛型完整类型化、错误类 instanceof、无运行时校验库 | Filled |

---

## 如何阅读与使用

1. 本包源码侧约定见同目录 `backend/`；`frontend/` 只讲 `server/` 如何正确消费。
2. 消费方改动（如改 `ProjectFileIndexAdapter`）同时参考 `.trellis/spec/guides/cross-layer-thinking-guide.md`，因为涉及 cache 内核 ↔ 领域 adapter ↔ DTO 三层数据流。
3. 报告「组件」「hook」「状态管理」等词在本文中都按**消费契约的等价物**理解，不要按 Web 前端语义照搬。

---

**语言**：本目录使用简体中文书写；代码标识符、路径、命令原文保留不翻译（与 `AGENTS.md` 一致）。
