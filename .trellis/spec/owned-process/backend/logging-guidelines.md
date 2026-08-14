# Logging Guidelines

> 本包不打印日志。本文记录「无日志」的现状、结构化错误字段约定，以及调用方如何记录。

---

## Overview

`@notnotype/owned-process` 的 `src/` 没有任何 `console.log` / logger 调用，也没有引入日志依赖。这是刻意设计：目标的 stdout/stderr 是 Supervisor 协议的边界（`pipe`/`inherit` 都继承监督器自己的 fd，由 Adapter 连接给宿主），库自身写入 stdout 会污染产品日志与协议。

模板问题的实际答案：

- 用什么日志库？→ 无。
- 日志级别与何时使用？→ 不适用。
- 应该记录什么？→ 错误信息通过 `OwnedProcessError` 的 `stage` / `osError` / `cause` 结构化字段传播，由调用方决定是否记录。
- 不应记录什么？→ 本包不接触 token / argv / env 的日志面；消费方遵循 AGENTS.md 的结构化日志与最小诊断原则。

## Log Levels

不适用：本包不产生日志级别。需要区分严重程度的是消费方（server/、desktop/ 等），它们按 AGENTS.md 使用结构化日志（例如 `this.logger.debug({ kind: message.kind }, "...")`）。

## Structured Logging

本包的结构化载体是 **错误对象字段** 而非日志行：`OwnedProcessError` 固定携带 `stage`（机器可读平台阶段）与可选 `osError`（Win32 错误码）：

```ts
// packages/owned-process/src/windows-adapter.ts
supervisor.once("error", (error) => beginFailure(new OwnedProcessError(
    `无法启动Windows自有进程监督器：${error.message}`,
    {stage: "supervisor-spawn", cause: error},
)));
```

消费方记录时保留这些字段（AGENTS.md：「日志使用结构化字段和自然语言消息」）。`packages/owned-process/tests/windows-release-smoke.ts` 是包内唯一的 `console.log` 先例，且只输出单行 JSON 状态用于 Release 门禁：

```ts
console.log(JSON.stringify({status: "passed", bash, runtime: process.execPath}));
```

## What to Log

由调用方记录，例如：

- `runBash` 的 timeout / abort 分类结果（`server/agent/tools/file-tools.ts` 抛出的领域错误）。
- Electron spike 的启动阶段与失败（`desktop/spikes/electron/src/main.ts` 的 `setSplashStage` / `console.error`）。
- 消费方 `void lease.terminate(reason).catch(() => undefined)` 时，决定是否上报（多个消费方先例）。

## What NOT to Log

- 本包/消费方不记录目标进程的完整 argv、环境变量、shutdown token 或正文。先例：Task 117 的 `runtime.lease` 只保存最小版本化诊断 JSON（lease ID、PID、时间、Bun/Node 版本），明确「不记录 argv、环境、token、cwd 或正文」（`docs/tasks/117-windows-process-tree-lifecycle/README.md`）。
- 不把目标 stdout/stderr 误当日志：它们是产品输出，按 `OwnedProcessSpec.stdio` 合同处理，不进入本包的日志面。
- 不记录 PII / 密钥：本包无此类数据面，消费方同样遵守仓库规则。
