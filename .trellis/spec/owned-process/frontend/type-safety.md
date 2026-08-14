# Type Safety

> 消费 `@notnotype/owned-process` 的类型约定：类型组织、消费契约类型与运行时验证。

---

## Overview

模板里的类型问题在本包的真实对应：

- 用什么类型系统？→ TypeScript strict（`packages/owned-process/tsconfig.json`：`"strict": true`）。
- 类型如何组织？→ 领域类型集中在 `packages/owned-process/src/types.ts`，`src/index.ts` re-export；消费方 `import type`。
- 用什么验证库？→ 无 Zod 等；监督协议用 Adapter 内手写运行时校验（`parseSupervisorMessage`）+ 类型守卫。
- 如何利用类型推断？→ 消费方从 `spawnOwnedProcess` 返回值推断 `OwnedProcessLease`；终态用可辨识的 `terminationReason` 联合类型。

## Type Organization

- 库侧：`types.ts` 定义 `OwnedProcessSpec` / `OwnedProcessLease` / `OwnedProcessCompletion` / `OwnedProcessStdio` / `OwnedProcessTerminationReason` / `OwnedProcessError`；`index.ts` 统一 re-export，消费方只从包名导入：

```ts
// desktop/spikes/electron/src/main.ts
import {
    spawnOwnedProcess,
    type OwnedProcessCompletion,
    type OwnedProcessLease,
} from "@notnotype/owned-process";
```

- 消费侧：`server/agent/tools/file-tools.owned-process.test.ts` 只 `import type {OwnedProcessTerminationReason}`；`shared/source-dev-launcher.test.ts` 只 `import type {OwnedProcessCompletion, OwnedProcessLease}`。

## Validation

- 库内：监督协议消息是外部未知数据，Adapter 用 `unknown` 入参 + 逐字段 `typeof` 校验，失败抛 `OwnedProcessError`（`stage: "protocol"`，见 backend/error-handling.md）。
- 终止原因白名单用类型守卫：`isTerminationReason(value): value is OwnedProcessTerminationReason`（`posix-adapter.ts` / `windows-adapter.ts` 各一份）。
- 消费侧：不做运行时校验；类型契约即验证。跨层投影时保持类型：`productExit` 返回显式 `{code: number | null; signal: string | null}`。

## Common Patterns

- 类型守卫投影：`shared/source-dev-launcher.test.ts` 的 `lease()` 返回 `OwnedProcessLease & {terminate: ReturnType<typeof vi.fn>}`，在测试中保留类型的同时注入 mock。
- 可辨识联合：`SupervisorMessage` 按 `kind` 区分 `ready` / `complete` / `terminated` / `error`（库内）。
- 可选字段语义：`terminationReason` 仅在主动终止时非空，消费方用 `if (completion.terminationReason === "timeout")` 判别。

## Forbidden Patterns

- `any`：消费方与库都禁止；外部未知数据用 `unknown` + 校验。
- 无理由类型断言 `as`：监督消息校验通过后的收窄断言是唯一先例，且紧邻校验代码（`posix-adapter.ts` 的 `candidate.signal as NodeJS.Signals | null`）。
- 在消费方重新声明库类型：从 `@notnotype/owned-process` `import type`，不要复制结构。
- 把 `OwnedProcessCompletion` 强转成别的形状：用投影函数显式转换（`productExit` 先例），保持类型完整。
