# Quality Guidelines

> 消费 `@notnotype/owned-process` 的代码质量约定：消费方测试模式与禁止模式。

---

## Overview

质量门槛与仓库一致：typecheck + 相关 vitest 测试。本包的消费契约是复杂、容易回归的路径，消费方必须有测试（AGENTS.md：「对外合同、复杂逻辑和容易回归的路径补充测试」）。测试通过 `vi.mock("@notnotype/owned-process")` 隔离库，不启动真实进程。

模板问题的实际答案：

- 禁止哪些模式？→ 见 Forbidden Patterns。
- 强制哪些 lint 规则？→ 无 lint 工具；同 backend/quality-guidelines.md 的说明。
- 测试要求？→ 见 Testing Requirements。
- 代码审查标准？→ 见 Code Review Checklist。

## Forbidden Patterns

- 在消费方测试里真实 spawn 本包（需要验证库本身时用包的集成测试，见 backend/quality-guidelines.md）：消费方测试 mock 库。
- 消费方代码做 `process.platform === "win32"` 分支来推断进程行为：平台细节由库隐藏。
- 忽略 `terminate()` 失败：可预期失败用 `.catch(() => undefined)` 显式处理，不允许 unhandled rejection。
- 无限等待 `completion`：跨层协议需要额外窗口时用 `Promise.race` 加超时（electron `shutdown` 30s 窗口先例）。
- 把库的 `OwnedProcessError` stage 当用户文案直接展示：消费方要映射为领域文案。

## Required Patterns

- 消费方测试用 `vi.hoisted` + `vi.mock` 固定库的返回，再断言 `spawnOwnedProcess` 收到的 spec 与 `terminate` 调用：

```ts
// server/agent/tools/file-tools.owned-process.test.ts
const ownedProcess = vi.hoisted(() => ({spawn: vi.fn()}));
vi.mock("@notnotype/owned-process", () => ({
    spawnOwnedProcess: ownedProcess.spawn,
}));
```

- fake timers 验证 timeout 与 abort 的时序竞态（`vi.useFakeTimers()` + `vi.advanceTimersByTimeAsync`，见 `file-tools.owned-process.test.ts`）。
- 用 deferred promise 控制 `completion` 的 resolve 时机，验证竞态终态（`shared/source-dev-launcher.test.ts` 的 `deferred<T>()`）。
- 断言 `terminate` 调用 reason 序列（如 `["timeout", "abort"]`），锁定终止语义。

## Testing Requirements

- `server/agent/tools/file-tools.owned-process.test.ts`：runBash 的 timeout/abort 映射。
- `server/agent/tools/file-tools.output-cleanup.test.ts`：进程失败时 OutputAccumulator 清理。
- `shared/source-dev-launcher.test.ts`：runSourceDev 的 graceful/forced shutdown、退出码传播、幂等 terminate。
- `packages/neuro-book-manager/src/app-commands.test.ts`：Manager 侧 mock 消费。
- 真实端到端（真实 Bash 收口、Windows Portable）由包的集成测试与 `bun run test:windows-owned-process` 门禁承担，不在消费方单测里重复。

## Code Review Checklist

- [ ] 消费方导入的是包名 `@notnotype/owned-process`，不是 `#owned-process/*`（后者只限包内）。
- [ ] spec 参数显式、类型完整；stdin/stdout/stderr 策略与产品协议一致（如 Manager 独占 stdin 用 `"ignore"`）。
- [ ] 每条 terminate 路径都有界且处理 rejection。
- [ ] completion 的 `terminationReason` 被映射为领域结果，没有泄漏库内部 stage。
- [ ] 竞态（timeout 同时 abort、shutdown 同时强杀）有测试锁定。
- [ ] typecheck 与聚焦测试已实际运行。
