# Database Guidelines

> 本包没有数据库层。本文记录「无数据库」的现状，以及实际的状态管理机制（进程句柄与 lease 管理）。

---

## Overview

`@notnotype/owned-process` 不读写数据库，也不依赖任何 ORM / 迁移工具。它管理的外部状态是 **OS 进程句柄**：POSIX 的独立 process group 与 Windows 的 Job Object。每个 `spawnOwnedProcess()` 调用产生一个 `OwnedProcessLease`，其生命周期状态收敛为单一 `completion` promise，而不是持久化记录。

模板问题的实际答案：

- 用什么 ORM/查询库？→ 无。
- 迁移如何管理？→ 无（`prisma` / `scripts/db/` 属于仓库其他部分，不适用于本包）。
- 表/列命名约定？→ 无表。
- 事务如何处理？→ 无事务；等价概念是「终态只提交一次」的 settle/rejectOnce 幂等提交（见 error-handling.md）。

## Query Patterns

本包没有查询模式。等价的实际模式是「进程生命周期状态机」：

- 状态字段：`settled`、`terminalMessage`、`terminalError`、`terminationReason`、`watchdog`（见 `packages/owned-process/src/posix-adapter.ts`）。
- 有界等待：所有等待都有 watchdog，不无限挂起：

```ts
// packages/owned-process/src/posix-adapter.ts
function armWatchdog(message: string, waitMs: number): void {
    if (watchdog || settled) return;
    watchdog = setTimeout(() => rejectOnce(new OwnedProcessError(message, {
        stage: "hard-kill-wait",
        cause: terminalError,
    })), waitMs);
}
```

- 状态收敛：监督进程的 `ready` / `complete` / `terminated` / `error` 消息最终收敛为 lease 的单一 `completion` promise（成功 resolve 或失败 reject）。

## Migrations

不适用：本包无 schema、无持久化数据。新增监督协议字段或类型字段不需要迁移，但必须同步更新 `src/*-adapter.ts` 的 `SupervisorMessage` 联合类型与 `parseSupervisorMessage` 运行时校验（类型即契约，参考 `../guides/cross-layer-thinking-guide.md`）。

## Naming Conventions

- 无表名/列名约定。适用的命名约定见 directory-structure.md（`OwnedProcess*` 类型前缀、`<platform>-adapter.ts` 文件命名）。
- 状态字段使用 `settled` / `terminal*` / `terminationReason` 等描述性名称，保持两个 Adapter 一致（`posix-adapter.ts` 与 `windows-adapter.ts` 字段名一一对应，便于对照审查）。

## Common Mistakes

- 把进程管理当数据库或进程注册表：Task 117 明确拒绝按进程名/命令行/ParentProcessId 扫描或写 PID 文件来「查询」进程（见 `docs/tasks/117-windows-process-tree-lifecycle/README.md` 的 Rejected Approaches）。所有权必须在 spawn 之前建立（Windows 先建 Job 再启动目标），不能事后枚举。
- 在包内引入持久化或全局状态：本包每次调用都是独立 lease，不跨 invocation 保留状态；监督器不写 PID 文件、不重绑。
- 把其它层（`scripts/db/`、Prisma）的数据库约定误带入本包：本包无数据库，数据库相关改动应放在消费方（server/ 等）。
