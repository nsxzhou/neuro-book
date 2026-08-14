# Component Guidelines

> 本包没有 UI 组件。本文记录消费方如何把 `OwnedProcessLease` 组合进自己的流程（组件组合的等价物）。

---

## Overview

本包不提供 Vue/React 组件；消费方是 server 工具、Electron 启动器和 CLI。模板里的组件问题在本包的真实对应：

- 组件模式是什么？→ 消费方函数把 `lease`（stdin/stdout/stderr + completion + terminate）作为可组合单元嵌入流程。
- props 如何定义？→ 启动参数用 `OwnedProcessSpec` 类型化对象。
- 如何组合？→ 见 Component Structure。
- 无障碍标准？→ 不适用（无 DOM）。

## Component Structure

消费方函数的典型结构（以 `server/agent/tools/file-tools.ts` 的 `runBash` 为例）：

1. 构造 `OwnedProcessSpec`（command/args/cwd/env/stdio/graceMs/hardKillWaitMs）。
2. 调用 `spawnOwnedProcess(spec)` 拿 `lease`。
3. 挂 `lease.stdout` / `lease.stderr` 的 `data` 监听，按需转发。
4. 把 timeout timer 与 `AbortSignal` 接到 `lease.terminate(reason)`。
5. `await lease.completion`，把 `terminationReason` / `exitCode` 映射为领域结果。

```ts
// server/agent/tools/file-tools.ts (runBash 骨架)
const lease = spawnOwnedProcess({
    command: input.bash,
    args: ["-lc", command],
    cwd: input.cwd,
    env: input.env,
    stdout: "pipe",
    stderr: "pipe",
    windowsHide: true,
    graceMs: 250,
    hardKillWaitMs: 3_000,
});
const onAbort = () => {
    acceptsOutput = false;
    void lease.terminate("abort").catch(() => undefined);
};
if (input.timeout !== undefined && input.timeout > 0) {
    timeoutHandle = setTimeout(() => {
        acceptsOutput = false;
        void lease.terminate("timeout").catch(() => undefined);
    }, input.timeout * 1000);
}
```

## Props Conventions

启动参数一律用 `OwnedProcessSpec` 字面量显式给出，不隐式依赖默认值。两个真实例子：

- `server/agent/tools/file-tools.ts`（runBash）：`stdin` 省略（默认 `"ignore"`），`stdout/stderr: "pipe"`，`graceMs: 250`，`hardKillWaitMs: 3_000`。
- `desktop/spikes/electron/src/main.ts`（launchProduct）：`stdin: "pipe"`（需要写 NDJSON 控制面），`stdout/stderr: "pipe"`，`graceMs: 1_000`，`hardKillWaitMs: 5_000`。
- `packages/neuro-book-manager/src/app-commands.ts`（Product 启动）：`stdin: "ignore"`（Manager 独占宿主 stdin，Windows 上同时读取并继承同一 pipe 会阻塞 Product 启动，源码注释写明），`stdout` 可配置。

## Styling Patterns

不适用：无 UI、无样式。

## Accessibility

不适用：无 DOM 可访问性要求。等价的「用户可感知」要求是：终止流程必须有界且结果确定（Task 117 Goal），消费方不得让 UI 返回而进程留在后台。

## Common Mistakes

- 消费方自己实现 spawn→kill 收口，绕过本包：Task 117 根因就是这个反模式，必须使用 `spawnOwnedProcess`。
- 在 `data` 回调里抛错导致 unhandled error：消费方用 `acceptsOutput` 之类的标志在 timeout/abort 后丢弃输出（`runBash` 先例）。
- 忘记处理 `lease.terminate(...)` 的 rejection：用 `void lease.terminate(reason).catch(() => undefined)` 吞掉可预期的失败（多个消费方先例）。
- 只 await completion 而不接 terminate：超时/关闭时进程树不会被收口。
