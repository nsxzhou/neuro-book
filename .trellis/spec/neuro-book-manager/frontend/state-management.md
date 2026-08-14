# State Management

> 本包状态的持久化合同与流转约定（无前端框架状态）。

---

## Overview

本包没有 Vue/Pinia/React 全局状态，也没有"服务端状态"缓存层。状态全部是**磁盘上的 JSON 合同**：读取后严格校验，修改走原子写入与 Operation Journal。三层状态：

1. 用户级配置 `~/.neuro-book-manager/config.json`（`src/manager-config.ts`，typebox `ManagerConfigSchema`）；
2. 每实例 Installation Manifest `<root>/.deploy/installation.json`（`src/manifest-store.ts` / `src/schema.ts` 的 `parseInstallationManifest`）；
3. 崩溃恢复账本 Operation Journal（`src/operation.ts`，`planned -> applied`，禁止倒退）。

回答模板问题：

- 状态管理方案：无框架；持久化 JSON + schema 校验 + 原子写；
- 本地 vs 全局：无内存全局状态；"全局"指用户级 config，实例状态按 Installation Root 隔离；
- 服务端状态：Release Manifest 从 GitHub API 拉取后严格解析，不落缓存；
- 派生状态：status / doctor / preflight 报告由 manifest + 运行时探测派生，每次重新计算（`src/installation-health.ts`、`src/install-preflight.ts`）。

---

## State Categories

| 类别 | 位置 | 读写入口 |
|------|------|----------|
| 用户级配置 | `~/.neuro-book-manager/config.json` | `src/manager-config.ts`（`readManagerConfig` / `registerManagerInstance`） |
| 实例状态 | `<root>/.deploy/installation.json` | `src/manifest-store.ts`（`readInstallationManifest` / `writeInstallationManifest`） |
| 操作账本 | `<root>/.deploy/` 下 journal（经 `operation.ts`） | `src/operation.ts`（`createOperation` / `setOperationEffect`） |
| Release 元数据 | GitHub API（运行时拉取） | `src/manifest-store.ts` 的 `resolveReleaseManifest` |

用户级配置的写入契约示例（`src/manager-config.ts`）：

```ts
const ManagerConfigSchema = Type.Object({
    schemaVersion: Type.Literal(1),
    defaultInstanceId: Type.Union([Type.String({minLength: 1}), Type.Null()]),
    preferences: Type.Object({
        channel: Type.Union([Type.Literal("stable"), Type.Literal("canary")]),
        installDirectory: Type.String({minLength: 1}),
        discoveryRoots: Type.Optional(Type.Array(Type.String({minLength: 1}))),
    }, {additionalProperties: false}),
    instances: Type.Array(Type.Object({
        id: Type.String({minLength: 1}),
        name: Type.String({minLength: 1}),
        root: Type.String({minLength: 1}),
        registeredAt: Type.String({pattern: ISO_DATE_PATTERN}),
        lastUsedAt: Type.String({pattern: ISO_DATE_PATTERN}),
    }, {additionalProperties: false})),
}, {additionalProperties: false});
```

读取后立即校验：`if (!Value.Check(ManagerConfigSchema, value)) throw new Error(...)`（`src/manager-config.ts`）。

---

## When to Use Global State

本包没有内存全局状态。等价决策是"数据放哪个 JSON 合同"：

- 跨实例共享的用户偏好 / 索引 → 用户级 `config.json`；`managerConfigPath()` 支持 `NEURO_BOOK_MANAGER_CONFIG` 环境变量覆盖，测试与便携入口用它隔离（`src/manager-config.ts`）；
- 单实例部署事实 → Installation Manifest（Profile、组件、roots、版本），由 `parseInstallationManifest` 严格校验后才是可执行依据（`src/schema.ts`）；
- 崩溃恢复 / 回滚依据 → Operation Journal，必须 `planned` 先行、同 identity 写 `applied`，禁止状态倒退（`src/operation.ts`）：

```ts
if (previous?.state === "applied" && effect.state === "planned") {
    throw new Error(`Operation effect不能从applied退回planned：${effectIdentity(effect)}`);
}
if (effect.state === "applied" && !previous) {
    throw new Error(`Operation effect缺少planned intent：${effectIdentity(effect)}`);
}
```

---

## Server State

本包本身是"安装 / 运行时管理器"，没有客户端-服务端状态缓存。对应物：

- Release Manifest：每次操作按 channel/version 重新 `fetch` 并严格解析（`resolveReleaseManifest`，`src/manifest-store.ts`），不缓存到磁盘；
- 实例服务状态：`doctor` / `status` 每次运行时探测（进程、端口、HTTP 版本接口、Compose 镜像），派生报告不落盘（`src/installation-health.ts`）；
- Docker/Podman engine 选择：首次安装选定后写入 Installation Manifest，后续 start/update/doctor 始终用该 engine，不在 Docker 与 Podman 之间静默切换（仓库 README）。

关键约定（仓库 README 原文要点）：**调用前缓存的 Manifest 不能作为执行依据**；外置 heartbeat lease 取得后先恢复未完成 Operation，再重读磁盘 Manifest。

---

## Common Mistakes

- 用内存缓存代替重读磁盘：任何 mutating 命令前必须重读并重校验 manifest。
- 直接改 JSON 不校验 / 不原子写：必须 `parse*` 校验后走 `writeJsonAtomic`（`src/files.ts`，先写临时文件再 rename）。
- 跳过 Operation Journal 直接改文件：崩溃后无法恢复 / 回滚；journal 状态不允许 `applied` 退回 `planned`。
- 把用户偏好写进实例 Manifest，或把实例事实写进用户 config：两类状态各有合同，互不混用（`config.json` 只保存用户偏好、默认实例和实例目录索引）。
