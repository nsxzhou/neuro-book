# Backend Development Guidelines

> `@notnotype/file-snapshot-cache` 包的后端开发约定。

---

## Overview

本目录描述 `packages/file-snapshot-cache/` 的真实编码约定。该包是与业务领域无关的纯 TS 内存快照缓存内核：不扫描文件、不解释节点内容、不连接数据库、不输出日志，只负责并发构建去重、变更世代、稳定提交、watcher 生命周期、订阅和空闲回收。调用方（主要是 `server/workspace-files/`）负责构造完整 typed snapshot 与 watcher adapter。

所有约定都来自本包真实代码（`src/`、`tests/`、`benchmarks/`）与仓库根 `AGENTS.md`；没有先例的模式不写进本目录。

包对外入口是 `src/index.ts` 的 barrel 重导出，消费方从包名导入：

```typescript
import {SnapshotCache, SnapshotClosedError} from "@notnotype/file-snapshot-cache";
```


---

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | 扁平 src/ 布局、类型/并发/主类/桶文件职责 | Filled |
| [Database Guidelines](./database-guidelines.md) | 无数据库；内存 entry + generation/revision 世代 | Filled |
| [Error Handling](./error-handling.md) | `SnapshotClosedError` / `SnapshotUnstableError`、AbortSignal 传递、有界 diagnostics | Filled |
| [Quality Guidelines](./quality-guidelines.md) | strict TS、vitest、隔离测试、无 `any`、别名导入 | Filled |
| [Logging Guidelines](./logging-guidelines.md) | 本包不写日志；用 `diagnostics()` 暴露有界资源状态 | Filled |

---

## 如何阅读与使用

1. 以**本包现实**为准：每个约定都附 `packages/file-snapshot-cache/` 内的真实文件路径与代码片段。
2. 消费方（`server/workspace-files/`）侧约定见同目录 `frontend/`（本包无前端，`frontend/` 记录消费契约）。
3. 通用思考指南见 `.trellis/spec/guides/`，与本目录配合使用。

---

**语言**：本目录使用简体中文书写；代码标识符、路径、命令原文保留不翻译（与 `AGENTS.md` 一致）。
