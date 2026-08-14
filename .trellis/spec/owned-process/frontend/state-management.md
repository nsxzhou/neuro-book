# State Management

> 本包没有 UI 状态。本文记录真实的「进程生命周期状态」管理：lease、终态不可改写、消费方 shutdown 竞态。

---

## Overview

模板里的状态管理问题在本包的真实对应：

- 用什么状态管理方案？→ 无 Redux/Pinia 等。状态是每个 lease 的本地状态：`completion` promise + `terminate(reason)` 函数 + stdio 流。
- 本地 vs 全局状态如何划分？→ 全部是本地状态（每个 lease 独立）；本包不引入全局进程注册表。
- 如何处理 server state？→ 不适用；等价物是目标进程的终态 `OwnedProcessCompletion`。
- 派生状态模式？→ 消费方从 `completion` 派生自身状态（如 electron splash stage、source-dev 的 `signalCount`）。

## State Categories

- lease 状态（库内）：`settled`、`terminalMessage`、`terminalError`、`terminationReason`、`watchdog` —— 只存在于 `src/*-adapter.ts`，消费方不可见。
- 消费方状态：`acceptsOutput`（runBash）、`signalCount` / `shutdownPromise` / `forcedShutdownPromise`（runSourceDev）、splash stage（electron）。
- 终态 `OwnedProcessCompletion`：`exitCode` / `signal` / 可选 `terminationReason`；一旦 resolve 不可改写。

## When to Use Global State

不适用（无 UI 全局状态）。跨模块共享的不是状态而是契约类型：消费方之间通过 `OwnedProcessCompletion` 的类型投影传递终态（`scripts/cli/source-dev.ts` 的 `productExit()`），不共享 lease 对象。

## Server State

等价物是「进程终态缓存/传播」：

- `runSourceDev` 用 `lease.completion.then(productExit)` 投影为共享 shutdown 合同（`{code, signal}`）。
- `runBash` 不缓存状态：每次调用独立 lease。
- shutdown 竞态：首次信号走 graceful（HTTP shutdown），第二次信号立即 `lease.terminate("shutdown")` 强制收口；`terminate` 幂等保证只触发一次（`shared/source-dev-launcher.test.ts` 的「第二次信号幂等地立即强制收口」用例）：

```ts
// scripts/cli/source-dev.ts (runSourceDev 骨架)
const requestShutdown = (): void => {
    signalCount += 1;
    if (signalCount === 1) {
        shutdownPromise = shutdownNativeProduct({..., forceTerminate: async () => { await lease.terminate("shutdown"); }});
        return;
    }
    if (!forcedShutdownPromise) {
        forcedShutdownPromise = lease.terminate("shutdown").then(() => "forced" as const);
    }
};
```

## Common Mistakes

- 在消费方维护「进程已退出」的副本状态：终态只信 `completion`，否则会与库内状态不一致。
- 竞态下把结果重分类：`file-tools.owned-process.test.ts` 的「timeout请求到达但lease已自然完成时不把结果重分类」用例锁定——先到终态为准。
- 不处理 graceful 与 force 同时失败：`runSourceDev` 用 `AggregateError` 传播，不无限等待。
- 把 lease 塞进全局状态跨模块共享：lease 是单所有权对象，共享会造成双重 terminate/收口竞争。
