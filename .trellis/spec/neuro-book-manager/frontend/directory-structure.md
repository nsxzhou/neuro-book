# Directory Structure

> 本包（@notnotype/neuro-book-manager）源码组织约定。

---

## Overview

本包是 Bun ESM 包，无浏览器前端。源码全部位于 `packages/neuro-book-manager/src/`，采用"一个领域一个文件"的扁平布局：测试与源码同目录（`*.test.ts`），无 `components/`、`views/` 等前端目录。

回答模板问题：

- UI 代码位置：CLI 命令集中在 `src/cli.ts`，TUI 在 `src/tui.ts`，Clack 向导在 `src/install-guide.ts`；
- 模块组织：按领域拆文件（install / update / maintenance / runtime / tools / desktop / instances / manager-config）；
- 共享工具：`src/files.ts`（原子读写、SHA256）、`src/process.ts`（外部命令）、`src/paths.ts` / `src/root-locators.ts`（路径）、`src/error-message.ts`（错误文本）；
- 共享类型与校验：`src/types.ts`（领域类型）、`src/schema.ts`（typebox schema 与 parse/assert 函数）；
- 测试 fixture：`src/fixtures/`。

---

## Directory Layout

```
packages/neuro-book-manager/
├── README.md
├── package.json          # bin: neuro-book -> dist/neuro-book.mjs；imports: #manager/* -> ./src/*.ts
├── tsconfig.json         # strict；paths: #manager/*、nbook/*
├── vitest.config.ts      # include src/**/*.test.ts；environment node；testTimeout 20_000
├── scripts/
│   ├── build.mjs         # bun build 单文件 bundle -> dist/
│   └── pack-check.mjs
└── src/
    ├── neuro-book.ts     # 入口：#!/usr/bin/env bun + import "#manager/cli"
    ├── cli.ts            # commander 全部命令与参数解析（含错误边界 main()）
    ├── tui.ts            # blessed 多实例管理 TUI
    ├── install-guide.ts  # Clack 交互安装向导
    ├── error-message.ts  # formatCliError 错误边界文本
    ├── types.ts          # 共享领域类型（614 行）
    ├── schema.ts         # typebox schema + parse* / assert* 函数
    ├── config.ts         # 应用启动环境 / 鉴权
    ├── app-commands.ts   # 应用级操作：start / create-admin / migration
    ├── install-*.ts      # installer / install-preflight / installation-* / install-guide
    ├── update-*.ts       # updater / update-planner / update-preflight
    ├── maintenance.ts    # doctor / status 入口（委托 installation-health.ts）
    ├── operation.ts      # Operation Journal（planned -> applied，禁止倒退）
    ├── manifest-store.ts # installation.json / Release Manifest 读写
    ├── manager-config.ts # ~/.neuro-book-manager/config.json
    ├── files.ts          # 原子写入、SHA256、safeTarget
    ├── process.ts        # 外部命令 run / runCapture / runWithInput
    ├── platform.ts       # 平台支持矩阵
    ├── profiles.ts       # 六种 Install Profile 定义
    ├── blessed-static.ts # 静态引入 blessed widgets（避免顶层动态 require）
    ├── blessed-modules.d.ts # blessed 子路径 widget 的 ambient 声明
    ├── fixtures/         # 测试共享 fixture（runtime-image.ts 等）
    └── *.test.ts         # 与源码同目录的 vitest 测试
```

---

## Module Organization

- 新增功能按领域新建独立模块文件，模块名用 kebab-case：`install-preflight.ts`、`instance-discovery.ts`、`managed-asset-repository.ts`。
- 一个文件一个合同：`types.ts` 只放类型；`schema.ts` 只放 typebox schema 与 parse/assert；命令入口 `cli.ts` 只做参数解析与输出，业务逻辑委托给领域模块。
- 跨包共享合同用 `nbook/*` 导入仓库根，例如 `src/app-commands.ts`：

```ts
import {
    PRODUCT_RUNTIME_EXIT_CODE_AGENT_SESSION_STORE_LEASE_COMPROMISED,
    type ProductRuntimeCommandId,
} from "nbook/shared/product-runtime-contract";
```

- 测试 fixture 只放 `src/fixtures/`：`src/fixtures/runtime-image.ts` 被 `component.test.ts`、`schema.test.ts` 共用。

---

## Naming Conventions

- 文件：kebab-case（`install-preflight.ts`、`app-commands.ts`）；测试文件 `*.test.ts` 与源码同目录。
- 函数：动词前缀表达行为：`run*`（`runManagerTui`、`runInstallGuide`）、`inspect*`（`inspectInstallPreflight`、`inspectInstance`）、`parse*`（`parsePort`、`parseProfile`）、`assert*`（`assertInstallConsent`、`assertTool`）、`read*`/`write*`（`readManagerConfig`、`writeJsonAtomic`）、`format*`（`formatCliError`、`formatDoctor`）。
- 类型：`export type`，PascalCase；JSON 合同类型与 schema 同名（`InstallationManifest` / `InstallationManifestSchema`）。
- CLI 子命令：小写名词/动词，多级用父命令分组（`instances roots list`、`runtime install`、`tools path`、`admin create`）。

---

## Examples

- CLI 结构：`src/cli.ts`（`install`、`manage`、`instances`、`adopt`、`update`、`start`、`status`、`doctor`、`uninstall`、`desktop`、`runtime`、`tools`、`admin` 共 13 组命令）。
- TUI 结构：`src/tui.ts`（screen + header/list/detail + question/prompt + 键盘绑定）。
- 领域模块示例：`src/install-preflight.ts`（只读预检，无副作用）、`src/operation.ts`（journal 状态机）。
