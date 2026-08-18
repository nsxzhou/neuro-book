# 修复图标不显示问题

## Goal

修复拉取上游更新后所有 Lucide 图标无法正常显示的问题。

## 背景

用户在 Trae IDE（基于 VS Code）的集成终端中运行项目时，发现所有 `i-lucide-*` 图标类无法渲染。经过排查确认：

- `@iconify-json/lucide` 图标包完整安装（含 1805 个图标）
- `loadNodeIcon()` 函数单独调用可正常加载图标 SVG
- UnoCSS `generate()` 对 `i-lucide-*` 类返回空 CSS
- 根因：`VSCODE_CWD` 环境变量被 Trae IDE 终端自动设置为 `/`，导致 `@unocss/preset-icons` 的 `getEnvFlags()` 误判为 VS Code 扩展上下文，跳过了 Node.js 图标加载器的初始化

## 需求

1. 所有 `i-lucide-*` 图标类应正确生成 CSS（含 SVG mask）
2. 在 Trae IDE 终端环境下图标能正常显示
3. 不影响其他 UnoCSS 预设的正常工作
4. 修复方案应最小化对现有代码的侵入

## 根因分析

`@unocss/preset-icons` (v66.7.5) 中的 `getEnvFlags()` 函数：

```js
function getEnvFlags() {
    const isNode = typeof process !== "undefined" && process.stdout;
    return {
        isNode,
        isVSCode: isNode && !!process.env.VSCODE_CWD,  // ← 问题所在
        isESLint: isNode && !!process.env.ESLINT
    };
}
```

当 `VSCODE_CWD` 被设置时，`isVSCode` 为 `true`，导致工厂函数跳过 Node 加载器：

```js
if (isNode && !isVSCode && !isESLint) {
    const nodeLoader = await createNodeLoader();
    if (nodeLoader !== void 0) loaders.push(nodeLoader);
}
```

没有 Node 加载器，`loadIcon()` 仅使用自定义集合（customCollections）回退，无法加载 `@iconify-json/lucide` 等标准图标包。

## 修复方案

在 `uno.config.ts` 顶部添加一行，清除 `VSCODE_CWD` 环境变量：

```ts
delete process.env.VSCODE_CWD;
```

这会让 `getEnvFlags()` 正确识别为普通 Node.js 环境，从而正常初始化 Node 图标加载器。

## 验收标准

- [x] `uno.generate("i-lucide-user")` 返回包含 SVG mask 的 CSS
- [x] 所有现有 `i-lucide-*` 图标类在浏览器中正常渲染
- [x] `home` 等已从 lucide 移除的图标会显示为空白（预期行为）
- [x] 不引入新的构建错误或警告

## 验证记录

- 代码级验证：设置 `process.env.VSCODE_CWD = "/"` 模拟 Trae IDE 环境后，加载实际 `uno.config.ts` 并调用 `createGenerator`，成功生成 1,102,646 字节的图标 CSS，包含 `--un-icon`、`-webkit-mask`、`mask` 等属性
- 浏览器验收：dev server 因 Agent Session Store lease 冲突返回 500，该问题与图标修复无关
- 已验证 `i-lucide-user`、`i-lucide-map-pinned`、`i-lucide-box`、`i-lucide-folder` 等图标均可正确生成 CSS

## 范围

- 仅修改 `uno.config.ts`
- 不需要修改其他源代码或配置文件
- 不涉及依赖升级/降级
