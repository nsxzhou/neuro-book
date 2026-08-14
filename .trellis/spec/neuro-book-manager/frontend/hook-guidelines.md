# Hook Guidelines

> 本包无 React/Vue hooks；记录等价物与现状。

---

## Overview

本包没有 React/Vue，也没有自定义 hooks（`use*` 函数）。模板中的"hook"概念在本包不存在，现状是：

- 交互逻辑用普通 async 函数组织：Clack 向导 `src/install-guide.ts`、TUI 的 `refresh()` / `runAction()`（`src/tui.ts`）；
- 数据获取（外部进程、网络、磁盘）用领域模块函数：`src/process.ts` 的 `run` / `runCapture`、`src/manifest-store.ts` 的 `resolveReleaseManifest`、`src/download.ts` 的 `downloadVerified`；
- 没有 `use*` 命名约定，没有"共享有状态逻辑"抽象 —— 有状态数据全部持久化在 JSON 合同（见 state-management.md），函数本身无状态。

回答模板问题：

- 自定义 hooks：无；
- 数据获取：CLI/TUI 通过领域模块函数同步获取（进程、HTTP、磁盘），TUI 用 `refresh()` 重读全部状态；
- 命名约定：动词前缀（`run*` / `inspect*` / `read*` / `parse*` / `assert*`），见下；
- 共享有状态逻辑：无；状态在磁盘 JSON 合同，不在内存共享。

---

## Custom Hook Patterns

等价物是"领域模块导出普通函数"，不引入 class、框架抽象或 hook 命名。向导按步骤组织（`src/install-guide.ts`）：

```ts
const profile = await promptValue(p.select<InstallProfile>({
    message: "你希望怎样运行 NeuroBook？",
    initialValue: recommended,
    options: profileOptions(),
}));
const root = resolve(await promptValue(p.text({
    message: "安装到哪个目录？",
    initialValue: suggestedRoot,
    validate: (value) => value?.trim() ? undefined : "安装目录不能为空。",
})));
```

TUI 的"状态刷新"用 `refresh()`（重读配置与 manifest）加 `runAction`（串行化 + 错误呈现到详情面板）实现（`src/tui.ts`）：

```ts
const runAction = async (action: () => Promise<void>): Promise<void> => {
    if (busy) return;
    busy = true;
    try {
        await action();
    } catch (error) {
        detail.setContent(`{red-fg}操作失败{/red-fg}\n\n${error instanceof Error ? error.message : String(error)}`);
        screen.render();
    } finally {
        busy = false;
    }
};
```

---

## Data Fetching

本包没有 React Query/SWR。"数据获取"分三类，全部封装在领域模块，调用方不直接裸 fetch：

1. **外部命令**：`src/process.ts` 的 `run`（非零退出码抛异常）、`runCapture`（捕获 stdout/stderr）、`runWithInput`（通过 stdin 传 secret，不写日志、不放进 argv/env/错误消息）。
2. **网络**：`src/manifest-store.ts` 直接 `fetch` GitHub Release API；`src/download.ts` 的 `downloadVerified` 下载后校验 SHA256：

```ts
export async function downloadVerified(url: string, target: string, sha256: string): Promise<void> {
    const response = await fetch(url, {headers: {"User-Agent": "neuro-book-manager"}});
    if (!response.ok) {
        throw new Error(`下载失败 ${response.status}：${url}`);
    }
    await ensureDirectory(dirname(target));
    await writeFile(target, new Uint8Array(await response.arrayBuffer()));
    const actual = await sha256File(target);
    if (actual.toLowerCase() !== sha256.toLowerCase()) {
        throw new Error(`SHA256 校验失败：${basename(target)}，期望 ${sha256}，实际 ${actual}`);
    }
}
```

3. **磁盘 JSON**：`src/files.ts` 的 `readJson`（ENOENT 返回 null）配合 `src/schema.ts` 的 `parse*` / `assert*` 严格校验后再使用。

获取到的数据先校验再使用；调用前缓存的 Manifest 不能作为执行依据（仓库 README 明示，见 state-management.md）。

---

## Naming Conventions

没有 `use*` 命名。函数名用动词前缀表达行为（与 AGENTS.md 一致）：

- `run*`：执行操作（`runManagerTui`、`runInstallGuide`、`runWithInput`）；
- `inspect*`：只读探测（`inspectInstallEnvironment`、`inspectInstance`、`inspectUpdatePreflight`）；
- `read*` / `write*`：持久化读写（`readManagerConfig`、`writeJsonAtomic`）；
- `parse*` / `assert*`：校验（`parseProfile`、`parseInstallationManifest`、`assertInstallConsent`、`assertTool`）。

---

## Common Mistakes

- 为单点交互逻辑自创"hook / 组合式"抽象 —— 本包用普通函数，不为单点逻辑建抽象（AGENTS.md）。
- TUI 内长时间阻塞 —— 长操作（安装、更新、启动）退出 TUI 后继续（`src/tui.ts`）。
- 未校验就使用外部数据 —— 所有外部输入（Release Manifest、installation.json、config.json）必须先过 `src/schema.ts` 的 parse/assert。
- 把 secret 放进 argv/env/日志 —— 密码走 `readPasswordStdin`（stdin），secret 调用方不得把同一内容放进 argv、env 或错误消息（`src/process.ts` 注释原文）。
