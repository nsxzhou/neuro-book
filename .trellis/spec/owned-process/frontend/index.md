# Frontend Development Guidelines

> `@notnotype/owned-process` 的「frontend 层」约定。本包没有 Vue 组件、hooks 或 UI 状态；frontend 层指它被 `server/`、`desktop/`、`scripts/`、`packages/neuro-book-manager` 消费时的契约与消费模式。

---

## Overview

本包是纯后端库，但它服务于「面向用户的进程生命周期」：Agent Bash 工具、Electron 启动器、Source Dev launcher、Manager 的 Product 启动。这些消费方通过包名 `@notnotype/owned-process` 导入 `spawnOwnedProcess` 与类型，并各自把 `OwnedProcessLease` 组合进自己的启动/关闭流程。

本目录的指南不描述 Vue 组件约定（本包没有），而是记录：

- 消费契约：`spawnOwnedProcess` / `OwnedProcessLease` / `OwnedProcessCompletion` / `OwnedProcessTerminationReason`（详见 type-safety.md）。
- 消费模式：如何组合 lease、消费 stdio、映射终止原因、处理 shutdown 竞态（详见 component-guidelines.md / hook-guidelines.md / state-management.md）。

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | 消费方位置与导入路径 | Filled |
| [Component Guidelines](./component-guidelines.md) | 无组件；记录 lease 组合模式 | Filled |
| [Hook Guidelines](./hook-guidelines.md) | 无 hooks；记录 completion/stdio 消费模式 | Filled |
| [State Management](./state-management.md) | 无 UI 状态；记录 lease 生命周期与 shutdown 竞态 | Filled |
| [Quality Guidelines](./quality-guidelines.md) | 消费方测试与禁止模式 | Filled |
| [Type Safety](./type-safety.md) | 类型组织与消费契约类型 | Filled |

---

## How to Fill These Guidelines

本目录由 trellis bootstrap 填充，反映 2026-08 的真实消费方。若新增消费方或消费契约变化，同步更新 component-guidelines.md、state-management.md 与 type-safety.md；共享思考指南见 `../guides/`，不要改。

---

**Language**: 本指南使用简体中文书写；代码标识符、路径与命令保留原文。
