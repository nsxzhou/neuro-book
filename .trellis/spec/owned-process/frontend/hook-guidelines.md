# Hook Guidelines

> 本包没有 React/Vue hooks。本文记录消费方围绕 `lease.completion` 与 stdio 的「有状态逻辑」模式（hooks 的等价物）。

---

## Overview

模板里的 hooks 问题在本包的真实对应：

- 有哪些自定义 hooks？→ 无。等价物是消费方内部围绕 lease 的异步组合函数（`waitForSupervisor`、`runBash` 的 completion 处理、`runSourceDev` 的 shutdown 编排）。
- 数据获取如何处理？→ 目标进程的 stdout/stderr 是数据面：`lease.stdout` / `lease.stderr`（`Readable`）用事件监听消费；控制面走独立 IPC（库内部），消费方不接触。
- 命名约定？→ 无 `use*`；消费方函数用动词短语（`waitForSupervisor`、`requestShutdown`）。
- 如何共享有状态逻辑？→ 本包不共享 hook；消费方各持一个 lease，跨模块只共享类型契约。

## Custom Hook Patterns

消费方「等待启动就绪」的模式：把 `lease.completion` 与 stdout 行解析组合成一个 promise，同时监听双方错误，任一先到即失败（`desktop/spikes/electron/src/main.ts` 的 `waitForSupervisor`）：

```ts
// desktop/spikes/electron/src/main.ts (launchProduct 骨架)
const output = lease.stdout;
const reader = createInterface({input: output, crlfDelay: Infinity});
const ready = waitForSupervisor(reader, lease.completion, requestId, startupNonce, (error) => {
    void lease.terminate("background-verification-failure").catch(() => undefined);
    console.error(error);
});
```

失败路径必须显式 `lease.terminate(reason)` 收口，再向调用方抛错（`launchProduct` 的 catch 块先 `await lease.terminate("startup-failure")`）。

## Data Fetching

- 目标 stdout/stderr 用 Node `Readable` 事件消费；超时/abort 后要停止接受输出（`runBash` 的 `acceptsOutput` 标志）。
- 不需要轮询进程状态：终态由 `lease.completion` 单一 promise 提供，消费方不探测 PID。
- 有界等待由库内 watchdog 保证；消费方只在跨层协议需要额外窗口时用 `Promise.race`（electron `shutdown` 的 30 秒窗口先例）。

## Naming Conventions

- 消费方函数：`waitForSupervisor`、`requestShutdown`、`runSourceDev`、`runBash` —— 动词短语，不出现 `use`。
- 测试辅助：`lease(completion)` 工厂、`deferred<T>()`（见 `shared/source-dev-launcher.test.ts`）。

## Common Mistakes

- 把「等待 stdout 就绪」与「等待进程终态」混成一个无限 promise：要同时监听 stdout 行、`completion` 与错误，任一先到即 settle（`waitForSupervisor` 先例）。
- 消费方自行探测 PID 存活或扫描端口来判断收口：库的 `completion` 已收敛终态；消费方只需等它。
- 在 abort 后继续写入 stdin：`shutdown`/`repair` 流程先 `lease.stdin.end()` 再等待 completion（`desktop/spikes/electron/src/main.ts` 先例）。
