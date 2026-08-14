# Quality Guidelines

> `@notnotype/owned-process` 的代码质量约定：从 AGENTS.md 导入并校准，加本包特有禁止模式。

---

## Overview

质量门槛是：`bun run --cwd packages/owned-process typecheck`（`tsc --noEmit -p tsconfig.json`，`strict: true`）通过，`bun run --cwd packages/owned-process test`（vitest，`bun --bun` 运行）通过。仓库根没有 eslint 配置；类型检查与测试是门禁（`scripts/release/release-assets.test.ts` 校验 release 流程包含该 typecheck 命令，`.github/workflows/product-platforms.yml` 跑包测试）。

模板问题的实际答案：

- 禁止哪些模式？→ 见 Forbidden Patterns。
- 强制哪些 lint 规则？→ 无 lint 工具；由 AGENTS.md JS/TS 规则 + typecheck + code review 承担。
- 测试要求？→ 见 Testing Requirements。
- 代码审查标准？→ 见 Code Review Checklist。

## Forbidden Patterns

- 相对路径导入：必须用 `#owned-process/*` 别名（`packages/owned-process/src/posix-adapter.ts` 等全部遵循）。
- `any` / 无理由的类型断言：类型完整是硬性要求；只有监督消息这种外部未知数据才用 `unknown` + 显式校验（`parseSupervisorMessage`）。
- 按进程名 / 命令行 / ParentProcessId 扫描或写 PID 文件来收口进程：Task 117 明确拒绝（Rejected Approaches），所有权必须在 spawn 前建立。
- 只 kill 直接子进程、不拥有整棵树：Task 117 的根因（Windows Git Bash wrapper 退出不代表 MSYS 后代退出）。
- 用 `taskkill` / WMI 枚举回退：Windows 必须走 Job Object（Bun FFI），失败 fail closed，不自动回退。
- 无限等待：任何等待都必须有界（`armWatchdog`），不能只给 Promise 套一个外层 timeout 而让进程留在后台。
- 把平台特例泄漏给调用方：`spawnOwnedProcess` 是唯一分派点，调用方不做 `process.platform` 分支。
- 迟到事件改写终态：`settled` 检查是每个提交路径的前置条件。

## Required Patterns

- 4 空格缩进（全部源文件；`packages/owned-process/package.json` 也是 4 空格）。
- 类型完整：公开 API 全部显式类型；监督协议用可辨识联合（`SupervisorMessage`）+ `unknown` 入参运行时校验。
- 领域错误用 class（AGENTS.md「后端高领域逻辑使用 class」在本包校准为：领域错误类 `OwnedProcessError`；Adapter 保持函数式，因为本包无共享可变状态，函数式更贴合现状）。
- 结构化错误字段：`stage` / `osError` / `cause`。
- 幂等 `terminate(reason)`：重复调用返回同一个 promise（`packages/owned-process/src/posix-adapter.ts` 的 `if (terminationPromise) return terminationPromise;`）。
- 注释解释合同与非显然决策，用中文写；不为显然代码逐行注释（如监督源码中 stdio 边界的注释）。
- 测试覆盖对外合同、复杂逻辑和容易回归的路径。

## Testing Requirements

- 使用 vitest，包内独立配置：`bun --bun ../../node_modules/vitest/vitest.mjs run --config vitest.config.ts`；`packages/owned-process/vitest.config.ts` 设 `testTimeout: 20_000`（进程树收口是真实时序）。
- 测试文件命名 `*.test.ts`，放 `tests/`；fixture 是真实子进程脚本（`tests/fixtures/owned-root.ts` 等），不 mock 进程行为。
- POSIX-only 用例用 `it.runIf(process.platform !== "win32")` 跳过（`tests/posix-adapter.test.ts`）。
- Windows 协议单测用 `FakeSupervisor` 替身 + `vi.doMock("node:child_process")`（`tests/windows-adapter.test.ts`），验证消息形状与 close 时序，不依赖真实 Win32。
- 集成测试用真实 fixture 证明完整收口：孙进程退出 + TCP 端口可立即重新 bind（`tests/owned-process.test.ts` 的 `waitForProcessExit` / `bindPort`）。
- 消费方测试 mock 本包：`vi.hoisted` + `vi.mock("@notnotype/owned-process", () => ({spawnOwnedProcess: ...}))`（`server/agent/tools/file-tools.owned-process.test.ts`、`shared/source-dev-launcher.test.ts`）。
- Windows Release 门禁：`tests/windows-release-smoke.ts` 与 `bun run test:windows-owned-process`（`scripts/deploy/windows-owned-process-smoke.ts`）在真实 Portable Bun/Git Bash 上验证。

## Code Review Checklist

- [ ] 导入全部走 `#owned-process/*` 别名，无相对路径。
- [ ] 新公开 API 类型完整，无 `any`；监督消息有运行时校验。
- [ ] 终止路径幂等、有界（watchdog），终态只提交一次。
- [ ] 平台差异只出现在 `src/*-adapter.ts` / `src/*-supervisor-source.ts`，`index.ts` 之外无平台分支。
- [ ] 失败以 `OwnedProcessError`（带 `stage`）传播，没有把 ownership failure 伪装成成功。
- [ ] 相关测试已更新：协议单测 + 集成测试（真实 fixture）按风险匹配。
- [ ] typecheck 与包内 vitest 已实际运行（报告真实命令与结果）。
