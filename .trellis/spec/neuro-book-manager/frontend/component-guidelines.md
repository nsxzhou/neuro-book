# Component Guidelines

> 本包 UI"组件"的构建约定：CLI 命令、TUI widget、Clack 向导。

---

## Overview

本包没有 Vue/React 组件。"组件"在本包指三类用户界面单元，各有固定构建方式：

1. **CLI 命令**（commander）：`src/cli.ts` 中 `program.command(...).description(...).option(...).action(...)`；
2. **TUI widget**（blessed）：`src/tui.ts` 中用 `blessed.screen/box/list/question/prompt` 在 screen 上组装；
3. **交互向导步骤**（Clack）：`src/install-guide.ts` 中 `p.intro/select/text/note/confirm/spinner/outro`。

回答模板问题：

- 组件模式：命令对象 + 领域模块委托；TUI widget 直接在 screen 上组装；
- props：CLI 用 `.option()` 声明并逐个解析；TUI widget 用 options 对象；Clack 用选项数组；
- 组合：命令把业务委托给领域模块（`installer` / `updater` / `maintenance`），自身只做解析与输出；
- 可访问性：非 TTY 环境拒绝交互入口，并提供 `--json` 机器可读输出。

---

## Component Structure

CLI 命令的标准结构（`src/cli.ts`）：

```ts
program.command("install")
    .description("安装或接管 NeuroBook Installation Root；交互终端默认进入完整引导。")
    .option("--profile <profile>", `安装 Profile：${profileNames().join(", ")}`)
    .option("--yes", "使用默认值，不进入交互。", false)
    .option("--dry-run", "只打印操作计划。", false)
    .option("--json", "与--dry-run一起输出结构化预检和操作计划。", false)
    .action(async (options: {profile?: string; dir?: string; version?: string; releaseManifest?: string; channel?: ReleaseChannel; port?: number; auth?: boolean; yes: boolean; dryRun: boolean; json: boolean}) => {
        if (options.version && options.releaseManifest) throw new Error("--version 与 --release-manifest 不能同时使用。" );
        if (options.json && !options.dryRun) throw new Error("--json当前只与--dry-run一起使用。" );
        if (!options.dryRun) assertInstallConsent(options.yes);
        // ...委托给 install-preflight / installer / install-guide
    });
```

TUI 的标准结构（`src/tui.ts`）：先检查 TTY，再建 `screen`，按 top/left/width/height 摆放 box/list，注册键盘绑定，最后 `list.focus(); screen.render();`。

---

## Props Conventions

CLI 参数等价于 props：一律用 `.option()` 声明，并在 action 内用解析函数校验后使用。解析函数抛出带中文提示的 `Error`（`src/cli.ts`）：

```ts
function parsePort(value: string): number {
    const port = Number(value);
    if (!Number.isInteger(port) || port < 1 || port > 65535) throw new Error(`端口必须是 1-65535：${value}`);
    return port;
}
```

TUI widget 的 options 是普通对象字面量（`src/tui.ts`）：

```ts
const header = blessed.box({
    parent: screen,
    top: 0, left: 0, width: "100%", height: 3,
    tags: true,
    content: " {bold}NeuroBook Manager{/bold}  实例管理",
    style: {fg: "white", bg: "blue"},
});
```

Clack 向导用 `p.select` 的 `options` 数组描述可选项，每项含 `value` / `label` / `hint` / `disabled`（`src/install-guide.ts` 的 `profileOptions()`），并统一用 `promptValue()` 把 cancel symbol 转成"已取消操作"错误。

---

## Styling Patterns

- CLI 无样式；用户可见输出是普通文本或 JSON（`printJson` / `printObject`，`src/cli.ts`）。
- TUI 样式用 blessed `style` 对象（fg/bg/border）和 tags（`{bold}`、`{green-fg}`、`{gray-fg}`）。示例见 `src/tui.ts` 的 `formatDoctor()`：

```ts
const marker = check.status === "pass" ? "{green-fg}✓{/green-fg}" : check.status === "warn" ? "{yellow-fg}!{/yellow-fg}" : "{red-fg}✗{/red-fg}";
lines.push(`${marker} ${check.message}`);
if (check.remediation) lines.push(`  建议：${check.remediation}`);
```

- 状态色语义与 AGENTS.md 一致：成功/健康用绿（`{green-fg}`）、警告/待处理用黄（`{yellow-fg}`）、错误/阻断用红（`{red-fg}`）、选中/强调用蓝/青（style.bg/fg）。本包没有 Web 主题变量，blessed 颜色直接写在 style 对象。
- TUI 只静态引入实际使用的 blessed widgets（`src/blessed-static.ts`），不要 import blessed 顶层入口：顶层入口会动态 require 全部 widgets，无法形成可独立运行的 Bun 单文件 bundle（`src/blessed-static.ts` 注释原文）。

---

## Accessibility

- 所有交互入口先检查 TTY：`src/tui.ts` 与 `src/install-guide.ts` 都在开头检查 `process.stdin.isTTY` / `process.stdout.isTTY`，不满足时抛出带替代命令的 `Error`（例如"自动化安装请使用 neuro-book install --profile <profile> --yes"）。
- 非交互（自动化、管道）路径提供 `--json` / `--dry-run` 输出，不进入交互（`src/cli.ts` 的 `printJson`）。
- TUI 支持键盘与 vi 键位：widget 配置 `keys: true, vi: true, mouse: true`，并注册 `q` / `C-c` 退出（`src/tui.ts`）。

```ts
screen.key("r", () => void runAction(refresh));
screen.key(["q", "C-c"], () => {
    screen.destroy();
});
```

- 密码输入不 trim、不写日志、硬限制 4096 bytes（`readPasswordStdin`，`src/cli.ts`）。

---

## Common Mistakes

- 在非 TTY 环境直接跑 TUI / Clack 向导 —— 必须先检查 `process.stdin.isTTY`，并给出含非交互替代命令的可操作错误提示。
- 在 TUI 内同步执行长时间安装/更新 —— 本包约定是退出 TUI 后再继续（`src/tui.ts` 顶部注释："长时间运行的安装、更新和启动会退出 TUI 后继续"）。
- 交互输出与机器可读输出混用 —— 自动化场景只走 `printJson`，用户场景才输出中文提示。
- 引入未使用的 blessed widget —— 必须同步补 `src/blessed-static.ts` 与 `src/blessed-modules.d.ts`，保持单文件 bundle 可打包。
