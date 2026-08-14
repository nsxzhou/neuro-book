# Type Safety

> 类型系统、运行时校验与类型组织。

---

## Overview

回答模板问题：

- 类型系统：TypeScript `strict`（`tsconfig.json`），ESNext / Bundler resolution，包内别名 `#manager/*`，仓库根别名 `nbook/*`；
- 类型组织：`src/types.ts` 集中领域类型；`src/schema.ts` 集中 typebox schema 与 parse/assert；模块局部类型就近声明；
- 校验库：typebox（`Type` / `Value` / `Static`）做运行时 schema 校验；
- 类型推断：优先让 TS 推断；JSON 边界统一 `parse*` 把 `unknown` 收窄到领域类型。

---

## Type Organization

- 共享领域类型集中在 `src/types.ts`（614 行），例如 Profile 用字面量联合：

```ts
export type InstallProfile =
    | "source-dev"
    | "source-product"
    | "product-bun"
    | "windows-portable"
    | "source-docker"
    | "ghcr";
```

- 同一合同的运行时 schema 在 `src/schema.ts`，类型与 schema 一一对应（schema 是运行时真相，`types.ts` 的类型是 `Static` 派生或手工镜像）；模块局部类型就地声明（`src/tui.ts` 的 `InstanceView` / `ManagerListItem`，`src/install-guide.ts` 的 `InstallGuideDefaults`）。
- 外部库补类型用 ambient declaration：`src/blessed-modules.d.ts` 为 blessed 子路径 widget 声明 module（配合 `src/blessed-static.ts` 静态引入）。

---

## Validation

运行时校验统一用 typebox，模式是"schema 先校验 + 语义再校验"（`src/schema.ts`）：

```ts
export function parseInstallationManifest(value: unknown): InstallationManifest {
    assertSchema(
        InstallationManifestSchema,
        value,
        "installation.json 不符合 NeuroBook Manager schema v5；旧版安装必须重新安装，Windows Portable 只复用完整 data/。",
    );
    const manifest = value as InstallationManifest;
    assertSemVer(manifest.managerVersion, "managerVersion");
    assertInstallationSemantics(manifest);
    assertComponentPaths(manifest);
    return manifest;
}
```

schema 使用 `Type.Object(..., {additionalProperties: false})` 拒绝多余字段，并用 `Type.String({pattern: ...})` 约束格式（SHA256、revision、ISO 日期），例如 `src/schema.ts` 顶部的 `SHA256_PATTERN = "^[a-fA-F0-9]{64}$"`。业务 schema 用 `Type` + `Static` 派生类型（`src/app-commands.ts`）：

```ts
const ApplicationMigrationStepSchema = Type.Object({...}, {additionalProperties: false});
type ApplicationMigrationReport = Static<typeof ApplicationMigrationReportSchema>;
```

---

## Common Patterns

- 类型守卫用 `asserts`：`src/cli.ts` 的 `assertTool(value: string): asserts value is "rg" | "git"`；`src/profiles.ts` 的 `parseProfile` 在 `value in PROFILE_DEFINITIONS` 判断后收窄：

```ts
export function parseProfile(value: string): InstallProfile {
    if (value in PROFILE_DEFINITIONS) {
        return value as InstallProfile;
    }
    throw new Error(`不支持的安装 Profile：${value}`);
}
```

- 可选字段用 `Type.Optional` / `?`；可空用 `Type.Null()` 联合（如 `defaultInstanceId: Type.Union([Type.String({minLength: 1}), Type.Null()])`）。
- 判别联合表达状态：`src/tui.ts` 的 `type ManagerListItem = {kind: "registered"; view: InstanceView} | {kind: "discovered"; inspection: OfflineInspection}`；`src/types.ts` 的 `InstallationCheckStatus = "pass" | "warn" | "fail"`。
- 只读 / 执行输入用结构化 options 对象类型（`src/install-guide.ts` 的 `InstallGuideDefaults`、`src/installer.ts` 的 `InstallOptions`），不散传参数。

---

## Forbidden Patterns

- `any`（AGENTS.md：通常意味着设计需要重新讨论）；外部未知输入用 `unknown` 并在边界校验后收窄，代码旁注明原因。
- 绕过 schema 的 `as` 断言：`as InstallationManifest` 只允许出现在 `parse*` 内部、紧跟在 `assertSchema` 之后（如 `src/schema.ts`）。
- 无约束 `Record<string, unknown>` 到处传递：必须映射到具体领域类型。
- 为旧数据留兼容分支：项目处于快速开发阶段，按当前合同修改数据库和数据结构，不保留 `legacy` 命名与无必要的旧兼容分支（AGENTS.md）。
