# Frontend Development Guidelines

> 本包（@notnotype/neuro-book-manager）的用户界面层编码约定。本包没有浏览器前端，"frontend" 实际指 CLI/TUI 用户界面层。

---

## Overview

本目录记录 `@notnotype/neuro-book-manager` 的用户界面层约定。该层不是 Vue/浏览器前端，而是：

- commander 定义的 CLI 命令结构（`src/cli.ts`，入口 `src/neuro-book.ts`）；
- blessed 渲染的多实例管理 TUI（`src/tui.ts`）；
- Clack 交互式安装向导（`src/install-guide.ts`）；
- 用户可见错误消息与文案（`src/error-message.ts`，简体中文）。

入口链是 `src/neuro-book.ts` -> `src/cli.ts`（commander 解析与错误边界），`src/tui.ts` 在 `manage` 子命令下启动 blessed TUI：

```ts
// packages/neuro-book-manager/src/neuro-book.ts
#!/usr/bin/env bun
import "#manager/cli";
```

```ts
// packages/neuro-book-manager/src/cli.ts（节选）
const program = new Command()
    .name("neuro-book")
    .version(MANAGER_VERSION)
    .description("NeuroBook installation, runtime, toolchain and instance manager.")
    .enablePositionalOptions()
    .option("--root <path>", "指定命令操作的 NeuroBook Installation Root。")
    .option("--instance <name-or-id>", "指定用户级配置中注册的实例。")
    .showHelpAfterError();
```

模板中继承自 Vue/浏览器前端的章节在本包按"记录现实"改写为 CLI/TUI 语境：Component = CLI 命令与 TUI widget；Hook = 本包无此类抽象（说明现状并列出等价 helper 函数）；State Management = 用户级配置、Installation Manifest 与 Operation Journal 三层持久化 JSON 合同。

本包其余约定（目录结构、质量、类型安全）同样记录在下面各文件。所有 spec 以简体中文书写（与仓库根 AGENTS.md 一致），代码标识符、路径与命令原文保留不翻译。

---

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | src/ 扁平模块布局、测试同目录、`#manager/*` / `nbook/*` 别名导入 | Filled |
| [Component Guidelines](./component-guidelines.md) | CLI 命令（commander）、TUI widget（blessed）、Clack 向导的构建约定 | Filled |
| [Hook Guidelines](./hook-guidelines.md) | 无 React/Vue hooks；等价物为领域 helper 函数与 TUI refresh/runAction | Filled |
| [State Management](./state-management.md) | 持久化 JSON 合同：config.json、installation.json、Operation Journal | Filled |
| [Quality Guidelines](./quality-guidelines.md) | 禁止/必用模式、vitest 测试要求、评审清单 | Filled |
| [Type Safety](./type-safety.md) | TS strict + typebox 运行时校验、types.ts / schema.ts 组织 | Filled |

---

## How to Fill These Guidelines

每条约定都来自本包真实代码，并附真实文件路径与代码片段；找不到的模式明确说明现状，不编造理想做法。修改本 spec 时同步更新对应代码示例，保持"记录现实"。

---

**Language**: 简体中文（仓库根 AGENTS.md 默认语言；模板原文 "All documentation should be written in English" 与本仓约定冲突，按 AGENTS.md 覆盖）。
