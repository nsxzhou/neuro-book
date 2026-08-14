# Error Handling

> 本包的错误处理约定：结构化领域错误 + 单一终态提交。

---

## Overview

`@notnotype/owned-process` 不抛 HTTP 错误，也不返回错误码对象。它定义了一个领域错误类 `OwnedProcessError`，携带平台阶段（`stage`）与可选 OS 错误码（`osError`）；所有失败通过 `lease.completion` promise 的 reject 传播，且每个 lease 的终态只提交一次。

模板问题的实际答案：

- 定义哪些错误类型？→ `OwnedProcessError`（唯一错误类），见 `packages/owned-process/src/types.ts`。
- 错误如何传播？→ 监督进程消息 → Adapter 记录 `terminalError` → 等待监督进程 `close` 后 reject `completion`；IPC/协议失败同样先记录再在 `close` 后提交。
- 错误如何记录？→ 本包不打印日志；结构化字段（`stage` / `osError` / `cause`）随错误对象传播，由调用方记录（见 logging-guidelines.md）。
- 错误如何返回给客户端？→ 无客户端；调用方 await `lease.completion`，自行把 `OwnedProcessError` 或 `terminationReason` 映射为领域文案。

## Error Types

唯一的错误类型是 `OwnedProcessError`，构造时要求 `stage`，可选 `osError` 与 `cause`：

```ts
// packages/owned-process/src/types.ts
export class OwnedProcessError extends Error {
    readonly stage: string;
    readonly osError?: number;

    constructor(message: string, input: {stage: string; osError?: number; cause?: unknown}) {
        super(message, input.cause === undefined ? undefined : {cause: input.cause});
        this.name = "OwnedProcessError";
        this.stage = input.stage;
        this.osError = input.osError;
    }
}
```

`stage` 是平台阶段枚举字符串，真实取值见 `packages/owned-process/src/posix-adapter.ts` 与 `windows-adapter.ts`，例如 `supervisor-spawn`、`control-ipc`、`protocol`、`process-group-signal`、`process-group-probe`、`terminate-job`、`hard-kill-wait`。

## Error Handling Patterns

1. **监督进程错误先记录、close 后提交**：Adapter 收到 `kind: "error"` 消息时只保存 `terminalError` 并 arm watchdog，不立即 reject；等监督进程 `close` 确认句柄收口后才提交失败终态：

```ts
// packages/owned-process/src/windows-adapter.ts
supervisor.once("close", (code, signal) => {
    if (settled) return;
    if (terminalError) {
        rejectOnce(terminalError);
        return;
    }
    // ...
});
```

2. **终态只提交一次**：`settle` / `rejectOnce` 都先检查 `settled`，避免迟到事件改写结果：

```ts
// packages/owned-process/src/windows-adapter.ts
function rejectOnce(error: unknown): void {
    if (settled) return;
    settled = true;
    cleanup();
    rejectCompletion(error);
}
```

3. **watchdog 保证有界**：所有失败路径都 arm watchdog，超时以 `stage: "hard-kill-wait"` 的 `OwnedProcessError` 提交，杜绝无限等待（`armWatchdog` 见 database-guidelines.md）。

4. **协议消息运行时校验**：`parseSupervisorMessage` 对未知 `kind` / 字段形状抛 `OwnedProcessError`（`stage: "protocol"`），不静默接受。

## API Error Responses

没有 HTTP API。等价物是 `OwnedProcessCompletion` 中的 `terminationReason` 字段（仅主动终止时非空）与 `completion` 的 reject。消费方负责映射：

```ts
// server/agent/tools/file-tools.ts (runBash)
const completion = await lease.completion;
if (completion.terminationReason === "timeout") {
    throw new Error(`Command timed out after ${input.timeout} seconds`);
}
if (completion.terminationReason === "abort"
    || completion.terminationReason === "cancel"
    || completion.terminationReason === "shutdown") {
    throw new Error("Command aborted");
}
```

领域文案不进本包（Task 117 D2：Module 不把 `Command timed out` 等文案放进库）。

## Common Mistakes

- 把 ownership failure 伪装成成功 timeout：终止流程失败必须 reject `OwnedProcessError`（带 `stage` / `osError`），调用方不得假装已处理。
- 迟到事件改写终态：`close` / `exit` / 多条消息的组合顺序可能乱序到达；任何提交前都要检查 `settled`。
- 在 timer 回调里直接抛未捕获异常：watchdog / 轮询回调内的错误要捕获并转为 `OwnedProcessError` 提交（`packages/owned-process/tests/posix-adapter.test.ts` 的「进程组探测失败不会从 timer 回调抛出未捕获异常」用例锁定该行为）。
- 协议消息缺字段时静默接受：必须先通过 `parseSupervisorMessage` 校验，否则未知形状会绕过错误处理。
