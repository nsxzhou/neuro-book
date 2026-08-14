# Quality Guidelines

> 代码质量标准与验证要求。

---

## Overview

回答模板问题：

- 禁止模式：`any`、相对路径导入、绕过校验、非原子写入、无 TTY 同意执行安装、`--yes` 跳过 blocker、`bunx run`（详见 Forbidden Patterns）；
- lint：仓库无独立 ESLint 配置，靠 `tsc --noEmit` 严格模式 + 代码评审约束（AGENTS.md / CONTRIBUTING.md 开发规范）；
- 测试：vitest（bun 运行），`src/**/*.test.ts` 与源码同目录；复杂合同 / 跨进程 / 回滚路径必须有测试；
- 评审标准：见 Code Review Checklist。

---

## Forbidden Patterns

- `any` / 未校验的 `unknown` / 无约束的 `Record<string, unknown>`（AGENTS.md：`any` 通常意味着设计需要重新讨论）；外部未知输入必须在边界用 typebox schema 校验并说明原因。
- 相对路径导入：只用 `#manager/*`（包内）与 `nbook/*`（仓库根共享合同），见 `src/app-commands.ts` 的 `import {...} from "nbook/shared/product-runtime-contract"`。
- `bunx run @notnotype/neuro-book-manager`：仓库 README 明示 `bunx run` 会把包名按本地脚本或路径解析，Manager 不会被启动。
- 非交互执行不显式 `--yes`：`src/install-preflight.ts` 的 `assertInstallConsent` 强制；TTY 缺失不能被解释为同意默认安装。
- 让 `--yes` 跳过 blocker：仓库 README 明示 blocker 不能被 `--yes` 跳过（`--yes` 只接受"服务未启动"等 warning）。
- 直接拼接用户路径绕过 containment：必须走 `safeTarget`、`assertAbsolutePathWithin` 等路径边界。
- 非原子写 JSON / 文本：必须用 `writeJsonAtomic` / `writeTextAtomic`（`src/files.ts`，先写临时文件再 rename）。
- Operation Journal 状态倒退：`src/operation.ts` 禁止 `applied` 退回 `planned`。

---

## Required Patterns

- 4 空格缩进；ESM；TS `strict`（`tsconfig.json`）；不相对导入。
- 命令入口统一错误边界：`src/cli.ts` 的 `main()` 捕获后 `formatCliError(error)` + `process.exitCode = 1`：

```ts
async function main(): Promise<void> {
    try {
        if (process.argv.slice(2).length === 0) {
            await runContextEntry();
            return;
        }
        await program.parseAsync(process.argv);
    } catch (error) {
        p.log.error(formatCliError(error));
        process.exitCode = 1;
    }
}
```

- 用户可见错误经 `formatCliError` 递归展开 AggregateError 与 `Error.cause`（`src/error-message.ts`），避免恢复失败遮住最初原因。
- 用户可见文案：简体中文、1-2 句、写用户能做什么（AGENTS.md「面向用户的文字」），不出现内部类名 / 文件路径 / Task 编号。
- 类型与 schema 双重定义：类型在 `types.ts`，运行时校验在 `schema.ts`（typebox）。
- 输出区分人读 / 机读：`printObject` 给人、`printJson` 给 `--json` 机器消费（`src/cli.ts`）：

```ts
function printJson(value: object): void {
    console.log(JSON.stringify(value, null, 4));
}
```

---

## Testing Requirements

- 测试框架 vitest，用 Bun 运行：`bun --bun ../../node_modules/vitest/vitest.mjs run --config vitest.config.ts`（package.json `test` script）。
- 测试与源码同目录：`src/*.test.ts`（如 `src/error-message.test.ts`、`src/app-commands.test.ts`）。
- `vitest.config.ts`：`include: ["src/**/*.test.ts"]`、`environment: "node"`、`testTimeout: 20_000`（Manager 回归包含真实 Git、PowerShell 和子进程冷启动；共享 runner 负载下 5 秒不足以区分慢启动与挂死 —— 配置注释原文）。
- 外部边界用 `vi.mock` + `vi.hoisted` 隔离（`src/app-commands.test.ts`）：

```ts
const processCommands = vi.hoisted(() => ({
    capture: vi.fn(),
    run: vi.fn(),
    input: vi.fn(),
    available: vi.fn(),
}));
vi.mock("#manager/process", () => ({
    runCapture: processCommands.capture,
    run: processCommands.run,
    runWithInput: processCommands.input,
    commandAvailable: processCommands.available,
}));
```

- 真实文件系统测试用 `mkdtemp(join(tmpdir(), ...))` 建临时目录，`afterEach` 清理（`src/component.test.ts`）：

```ts
const roots: string[] = [];
afterEach(async () => {
    await Promise.all(roots.splice(0).map((root) => removePath(root)));
});
```

- 类型检查：`bun ../../node_modules/typescript/bin/tsc --noEmit -p tsconfig.json`（package.json `typecheck` script）。
- 验证命令：本包 `bun run test` / `bun run typecheck` / `bun run build`；提交 PR 时列出实际执行的命令，未执行的写"未运行"，聚焦测试通过不能写成全量测试通过（CONTRIBUTING.md）。

---

## Code Review Checklist

- [ ] 无 `any`、无未校验 `unknown`、无无约束 `Record<string, unknown>`。
- [ ] 只使用 `#manager/*` / `nbook/*` 导入，无相对路径。
- [ ] 每个外部输入（manifest / config / journal / Release）都经过 schema parse/assert。
- [ ] 写 JSON / 文本走原子写入；Operation Journal 无状态倒退。
- [ ] 交互入口有 TTY 检查；非交互路径有 `--json` / `--dry-run` 或强制 `--yes`。
- [ ] 用户可见错误经 `formatCliError`；中文文案符合「面向用户的文字」。
- [ ] 新增复杂合同 / 跨进程 / 回滚逻辑有同目录 vitest 测试且通过；`typecheck` 通过。
