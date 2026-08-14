# Quality Guidelines

> 本包的质量标准：strict TS + vitest + 包边界隔离测试；仓库当前没有 ESLint 配置，质量门禁是 `typecheck` 与测试。

---

## Overview

- TypeScript：`tsconfig.json` 开启 `strict: true`、`noEmit`、`moduleResolution: "Bundler"`、`target/module: ESNext`；类型检查命令 `bun run --cwd packages/file-snapshot-cache typecheck`（即 `tsc --noEmit -p tsconfig.json`）。
- 测试：vitest，`vitest.config.ts` 指定 `environment: "node"`、`include: ["tests/**/*.test.ts"]`、`testTimeout: 20_000`；命令 `bun run --cwd packages/file-snapshot-cache test`。
- 仓库当前**没有 ESLint/Prettier 配置**（本包与仓库根均无 `eslint.config.*` / `.eslintrc*`）；格式化以 4 空格缩进为事实标准（`src/` 全量 4 空格，无 tab、无 2 空格）。
- 基准：`bun run --cwd packages/file-snapshot-cache benchmark`（Node，`node --expose-gc --import tsx`）与 `benchmark:bun`（Bun），产物写入 `benchmarks/results/`，只测缓存编排与资源生命周期，不代表调用方扫描耗时。
- `AGENTS.md` 规则在本包的具体化：4 空格缩进、绝对路径别名（`#cache/*` / `#test/*`）、不用相对路径、高领域逻辑用 class、保持类型完整（无 `any`）、注释解释合同与非显然决策。

## Forbidden Patterns

- **相对路径导入**：包内 `import ... from "../src/..."` / `"./types"` 一律禁止；源码用 `#cache/*`、测试用 `#test/*`（`package.json` `imports` + `tsconfig.json` `paths` 双处维护）。
- **`any`**：`src/` 中无任何 `any`；外部未知数据只允许 `unknown`（仅出现在 catch 参数 `error: unknown` 与 `boundedDiagnosticError` 参数），并在代码旁说明原因。
- **向包内泄漏 NeuroBook 领域**：源码不得 import `nbook/`、`nuxt` / `h3` / `pinia` / `#imports`，不得出现 `ProjectSession`、`WorkspaceFileNode`、`sqlite|libsql|prisma`、`.nbook|.agent|frontmatter|workspace/`；`tests/isolation.test.ts` 逐条断言：

```typescript
// packages/file-snapshot-cache/tests/isolation.test.ts
expect(source).not.toMatch(/from\s+["']nbook\//);
expect(source).not.toMatch(/from\s+["'](?:nuxt|h3|pinia|#imports)/);
expect(source).not.toContain("ProjectSession");
expect(source).not.toContain("WorkspaceFileNode");
expect(source).not.toMatch(/(?:sqlite|libsql|prisma)/i);
```

- **生产依赖**：`package.json` 无 `dependencies`（devDependencies 只有 `@types/node`、`tsx`、`typescript`、`vitest`），隔离测试断言 `manifest.dependencies` 为 `undefined` 且脚本不依赖 monorepo 相对路径的 node_modules。
- **调用方恢复「先写入、再手动 invalidate」两段式协议**：本包 `mutate()` 自带串行与 generation 推进，调用方不得自行两段式。
- **无生产消费者的 projection/store 接口**：不留 `JsonProjectionStore` 之类死接口（隔离测试 `tests/isolation.test.ts` 第三个用例断言）。
- **隐式打开 watcher**：`read()` / `subscribe()` 不得隐式开 watcher；长期 owner 必须显式 `activate` 并等待 `ready`。

## Required Patterns

- 4 空格缩进；文件命名 kebab-case；`import type` 用于纯类型导入（`src/snapshot-cache.ts` 对 `FileSnapshot`、`SnapshotActivation` 等用 `import type {...} from "#cache/types"`）。
- 高领域逻辑使用 class：`SnapshotCache`（`src/snapshot-cache.ts`）、`AsyncSemaphore`（`src/concurrency.ts`）。
- 公开类型用 `readonly` 数组/字段表达不可变性（`FileSnapshot.nodes: readonly TNode[]`），builder 返回数组后所有权移交 cache，builder 不得继续修改。
- 并发边界：`AbortSignal` 贯穿 watcher/build/semaphore；每个 await 之后 `assertEntryOpen` 复查；同 key 共享单一 `buildPromise`；`AsyncSemaphore` 限制全局构建并发。

```typescript
// packages/file-snapshot-cache/src/concurrency.ts
async acquire(signal: AbortSignal): Promise<() => void> {
    if (signal.aborted) {
        throw signal.reason;
    }
    if (this.active < this.limit) {
        this.active += 1;
        return this.releaseOnce();
    }
    // 排队 waiter；abort 时从队列移除并 reject(signal.reason)
}
```

- 错误与诊断有界：错误文本经 `boundedDiagnosticError`（≤ 2_000 字符），每个 entry 只保留最近一次 build 失败。
- 关闭幂等：`close` / `closeAll` 通过 `closePromises` / `closeAllPromise` 去重；watcher close 失败时保留精确 handle 与 closed entry，后续调用重试同一资源。
- 公共契约用中文 JSDoc 注释解释（如 `types.ts` 每个接口一行契约说明），不为显然代码逐行注释。

## Testing Requirements

- 每个行为面有 vitest 用例，文件按模块命名：`concurrency.test.ts`、`snapshot-cache.test.ts`、`isolation.test.ts`。
- 竞态测试用 `tests/helpers.ts` 的 `deferred()`（手工完成 Promise）与 `waitFor()`（轮询条件）精确控制边界：

```typescript
// packages/file-snapshot-cache/tests/concurrency.test.ts
const queued = semaphore.acquire(queuedController.signal);
expect(semaphore.queuedCount).toBe(1);
queuedController.abort(new Error("cancelled"));
await expect(queued).rejects.toThrow("cancelled");
expect(semaphore.queuedCount).toBe(0);
```

- 参数化用例用 `it.each`（如 `it.each([1, 10, 100])("%i 个同 key cold reader 共享一个 build", ...)`）。
- 错误路径必须断言 typed 错误：`rejects.toBeInstanceOf(SnapshotClosedError)`、`rejects.toMatchObject({name: "SnapshotUnstableError", attempts: 2} satisfies Partial<SnapshotUnstableError>)`。
- 每次用例结束 `await cache.closeAll()`，避免 timer/subscriber 泄漏影响其他用例。
- 包边界自检是常规门禁的一部分（`isolation.test.ts` 三个用例），新增源码时必须保持通过。

## Code Review Checklist

- [ ] 无相对路径导入；源码只用 `#cache/*`，测试只用 `#test/*`；`tsconfig.json` 与 `package.json` 的别名两处同步。
- [ ] 无 `any`；`unknown` 仅用于外部未知输入并附说明；`import type` 用于纯类型。
- [ ] 不引入 `nbook/` / nuxt / sqlite 等领域依赖（`isolation.test.ts` 保持绿色）。
- [ ] 每个异步边界后 `assertEntryOpen`；late result 不会提交到 closed/新 incarnation entry。
- [ ] 公开 API 有 `readonly` 契约与中文 JSDoc；新增公开符号在 `src/index.ts` 重导出。
- [ ] build/mutation/watcher 失败有 diagnostics 覆盖且文本有界；关闭幂等可重试。
- [ ] 新增行为有聚焦 vitest 用例；涉及并发/世代的用例使用 `deferred` / `waitFor` 控制竞态。
- [ ] 不添加无生产消费者的接口/存储；不改 `package.json` 生产依赖。
