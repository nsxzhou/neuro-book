# Directory Structure

> `@notnotype/owned-process` 的模块组织与文件布局。

---

## Overview

本包是单一 workspace 包（`packages/owned-process/`），没有框架、没有数据库、没有 HTTP 层。代码按「平台 × 职责」组织：

- `src/`：库实现。公共入口、领域类型、平台 Adapter、监督进程源码生成器。
- `tests/`：vitest 测试与 fixture。测试用真实子进程（Node/Bash fixture），不用 mock 替代进程行为。

模板问题的实际答案：

- 模块/包如何组织？→ 平台差异隔离在 `*-adapter.ts` + `*-supervisor-source.ts` 一对文件；跨平台共享的领域类型集中在 `src/types.ts`；公共入口 `src/index.ts` 只做平台分派与类型 re-export。
- 业务逻辑在哪？→ 本包没有业务逻辑；领域逻辑是进程所有权状态机，分布在两个 Adapter 内（supervisor 协议解析、watchdog、终态提交）。
- API endpoints 在哪？→ 没有 API。唯一的公开入口是 `src/index.ts` 的 `spawnOwnedProcess(spec): OwnedProcessLease`。
- 工具/helper 如何组织？→ 模块私有函数与模块同文件定义（如 `parseSupervisorMessage`、`validWindow`、`isTerminationReason`），不做共享 utils 目录。

## Directory Layout

```
packages/owned-process/
├── package.json          # name: @notnotype/owned-process; imports 映射 #owned-process/*
├── tsconfig.json         # strict: true; paths: { "#owned-process/*": ["./src/*"] }
├── vitest.config.ts      # testTimeout: 20_000; alias #owned-process → ./src
├── src/
│   ├── index.ts                     # spawnOwnedProcess 平台分派 + 类型 re-export
│   ├── types.ts                     # OwnedProcessSpec/Lease/Completion/TerminationReason + OwnedProcessError
│   ├── posix-adapter.ts             # POSIX Adapter: process group 所有权
│   ├── windows-adapter.ts           # Windows Adapter: Job Object 所有权（Bun FFI）
│   ├── posix-supervisor-source.ts   # POSIX 监督进程源码生成器
│   └── windows-supervisor-source.ts # Windows 监督进程源码生成器
└── tests/
    ├── owned-process.test.ts        # 跨平台集成测试（真实 fixture 进程 + TCP 端口）
    ├── posix-adapter.test.ts        # POSIX 监督协议故障回归
    ├── windows-adapter.test.ts      # Windows Adapter 协议单测（FakeSupervisor）
    ├── windows-release-smoke.ts     # Windows Release 门禁 smoke（真实 Portable Bun/Git Bash）
    └── fixtures/                    # 真实子进程 fixture（owned-root / normal-exit-root / nested-owned-root 等）
```

## Module Organization

新增功能按以下规则落位：

- 跨平台公共类型或错误类 → `src/types.ts`。
- 新增平台所有权机制 → 在 `src/<platform>-adapter.ts` 实现 Adapter，并在 `src/<platform>-supervisor-source.ts` 实现对应监督进程源码；不改 `src/index.ts` 之外的公共签名。
- 修改监督协议（`kind` 消息形状）→ 同步修改 `src/*-adapter.ts` 的 `SupervisorMessage` 联合类型、`parseSupervisorMessage` 校验，以及 `src/*-supervisor-source.ts` 的发送方；两侧是同一协议的边界（参考 `../guides/cross-layer-thinking-guide.md` 的「Implicit Format Assumptions」）。
- 平台分派只在 `src/index.ts` 的 `spawnOwnedProcess` 出现，不允许业务调用方做 `process.platform === "win32"` 分支：

```ts
// packages/owned-process/src/index.ts
if (process.platform === "win32") {
    if (process.arch !== "x64") {
        throw new OwnedProcessError(`Windows Owned Process当前仅支持x64，实际为${process.arch}。`, {stage: "platform"});
    }
    return spawnWindowsOwnedProcess(spec);
}
return spawnPosixOwnedProcess(spec);
```

## Naming Conventions

- 文件：kebab-case（`posix-adapter.ts`、`windows-supervisor-source.ts`）。
- 平台 Adapter：`<platform>-adapter.ts`，导出 `spawn<Platform>OwnedProcess`。
- 监督源码生成器：`<platform>-supervisor-source.ts`，导出 `build<Platform>SupervisorSource` 与生产常量（`POSIX_SUPERVISOR_SOURCE` / `WINDOWS_SUPERVISOR_SOURCE`）。
- 类型：`OwnedProcess*` 前缀（`OwnedProcessSpec`、`OwnedProcessLease`、`OwnedProcessCompletion`、`OwnedProcessTerminationReason`）。
- 错误类字段：`stage`（平台阶段）、`osError`（可选 Win32 错误码）。
- 导入一律使用 `#owned-process/*` 别名，不用相对路径（AGENTS.md JS/TS 规则）。

## Examples

- 平台 Adapter 对：`packages/owned-process/src/posix-adapter.ts` 与 `packages/owned-process/src/windows-adapter.ts` —— 两者实现同一 `OwnedProcessLease` 契约，内部机制完全不同（process group vs Job Object）。
- 领域类型与错误：`packages/owned-process/src/types.ts`。
- 跨平台集成测试：`packages/owned-process/tests/owned-process.test.ts`（真实 fixture + TCP 端口证明完整收口）。
