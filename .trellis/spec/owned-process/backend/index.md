# Backend Development Guidelines

> `@notnotype/owned-process` 的后端开发约定。后端层指 `packages/owned-process/src/`（库实现）与 `packages/owned-process/tests/`（vitest 测试与 fixture）。

---

## Overview

`@notnotype/owned-process` 是 NeuroBook 的跨平台「自有进程树所有权」库：调用方声明启动参数与终止策略，库保证在 timeout、abort、cancel、shutdown、启动失败或宿主 IPC 断开时，完整进程树（含 MSYS/Git Bash 后代）在有界时间内被收口，并确认 stdio 与继承句柄释放。

后端实现由三部分组成：

- `src/types.ts`：领域类型（`OwnedProcessSpec` / `OwnedProcessLease` / `OwnedProcessCompletion` / `OwnedProcessTerminationReason`）与领域错误类 `OwnedProcessError`。
- `src/*-adapter.ts`：平台 Adapter，持有监督进程、解析监督协议、设置有界 watchdog，并把监督状态收敛为单一 `completion` promise。
- `src/*-supervisor-source.ts`：监督进程源码字符串生成器；Windows 用 Bun FFI + Job Object，POSIX 用独立 process group。

本指南把 AGENTS.md 的仓库级 JS/TS 规则按本包实际代码校准（4 空格缩进、`#owned-process/*` 别名导入、类型完整、结构化错误字段、vitest 测试），并记录本包特有的平台所有权约定。

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | 模块组织与文件布局 | Filled |
| [Database Guidelines](./database-guidelines.md) | 本包无数据库；实际机制是进程句柄与 lease 管理 | Filled |
| [Error Handling](./error-handling.md) | OwnedProcessError、stage/osError 字段、终态只提交一次 | Filled |
| [Quality Guidelines](./quality-guidelines.md) | 代码标准、禁止模式、vitest 测试要求 | Filled |
| [Logging Guidelines](./logging-guidelines.md) | 本包不打印日志；结构化字段通过 OwnedProcessError 传播 | Filled |

---

## How to Fill These Guidelines

本指南由 trellis bootstrap 填充，反映 2026-08 该包的真实实现。修改包的结构或约定后，请同步更新对应文件；新增平台 Adapter 或监督协议字段时，至少更新 directory-structure.md 与 error-handling.md。共享思考指南见 `../guides/`，不要改。

---

**Language**: 本指南使用简体中文书写；代码标识符、路径与命令保留原文。
