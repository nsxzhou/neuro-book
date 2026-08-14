# 左侧栏统一 Tooltip 提示系统

## Goal

创建小说进入 IDE 后，左侧栏（最左 48px 图标条 `NovelIdeActivityBar` + 工具面板头部 `NovelIdeToolPanel`）有大量纯图标按钮。当前依赖原生 `title` 提示，在 Tauri/WebView 桌面端不显示、Web 端延迟约 1s 且样式不统一。本次新增一个主题一致、悬停即时弹出的自定义 Tooltip，覆盖左侧整体所有图标按钮，让用户悬停即可知道按钮是干嘛的。

## Background（已确认事实）

- 左侧栏结构：`app/components/novel-ide/NovelIdeActivityBar.vue`（约 9–10 个纯图标按钮：书架/文件/角色/剧本/World Engine/请求记录/变更/AI 面板/账户/设置/More）与 `app/components/novel-ide/NovelIdeToolPanel.vue` 头部（上传单文件、上传项目、下载、同步资产、查看冲突、收起）。
- 现状：图标按钮普遍挂了原生 `:title`（文案已有 i18n），但项目同时发行 Electron 与 Tauri 桌面端，Tauri 的 WebView（macOS WKWebView / Windows WebView2）对原生 `title` tooltip 支持不可靠。
- 全项目没有自定义 Tooltip 组件/组合式函数；现有浮层范式为 `rounded-md border border-[var(--border-color)] bg-[var(--bg-panel)] shadow-xl`，主题变量见 `app/utils/theme/README.md`。
- `NovelIdeToolPanel` 的「收起」按钮（`i-lucide-minus`）当前完全无提示文案，需补 i18n。
- 约束：48px Activity Bar 是 Task 143 已验证的桌面契约，本次不改宽度；UI 只消费已有主题变量，不新增变量、不新增 Tailwind 色板。

## Requirements

- R1 新增通用 Tooltip 组件 `app/components/common/Tooltip.vue` 与定位工具 `app/utils/tooltip-position.ts`：
  - 触发槽包裹按钮，popover 通过 `Teleport` 挂到 body，避免被面板 `overflow-hidden` 裁剪。
  - 支持 `placement: "right" | "bottom"`，基于 `getBoundingClientRect` 手动定位并做视口钳制；不新增运行时依赖。
  - 行为：hover 约 300ms 显示 / 100ms 隐藏；键盘 focus 立即显示；滚动、点击、移出即隐藏；`role="tooltip"` + 触发元素 `aria-describedby`，保留触发元素 `aria-label`；尊重 `prefers-reduced-motion`（仅淡入，无位移）。
  - 视觉：只消费已有主题变量（`--bg-panel`、`--border-color`、`--text-main`、`--text-secondary`、`--shadow-color`），`rounded-md`、`px-2 py-1`、`text-xs`、`max-w-[220px]`、小箭头锚点。
  - disabled 按钮不触发 pointer 事件：Tooltip 监听包裹层（span）而非按钮本身，保证禁用态提示可弹出。
- R2 `NovelIdeActivityBar.vue`：primary/secondary 可见按钮、Agent 面板按钮、设置按钮、More 按钮套上 Tooltip（`placement="right"`）；移除被覆盖按钮的原生 `:title` 避免双提示；More 菜单项已有可见文字标签不加 Tooltip；账户按钮是真实菜单保持现状。
- R3 `NovelIdeToolPanel.vue`：头部「上传单文件 / 上传项目 / 下载 / 同步资产 / 查看冲突」按钮套上 Tooltip（`placement="bottom"`）；「收起」按钮补 i18n key `ide.toolPanel.collapsePanel`（zh「收起面板」/ en "Collapse panel"）并套上 Tooltip。
- R4 i18n：`app/i18n/locales/zh-CN.ts`、`en-US.ts` 各新增 1 条（收起面板）；其余全部复用现有 key，不新增描述文案。

## Acceptance Criteria

- [x] 悬停最左图标条每个图标按钮，约 300ms 在右侧弹出名称提示，样式与 8 套主题一致（实现完成；主题变量消费已核对，运行时观感待桌面端人工验收）。
- [x] 悬停工具面板头部按钮，在按钮下方弹出提示；「收起」按钮显示「收起面板」（实现完成；含头部「书架」按钮）。
- [x] 未打开 Project 时，悬停禁用的文件/剧本等按钮显示「名称 · 请先打开一个 Project」（实现完成；监听包裹层以覆盖 disabled 按钮，运行时待人工验收）。
- [x] 键盘 Tab 聚焦图标按钮时提示立即出现；开启「减少动态效果」时无位移动画（focus 立即显示与 `prefers-reduced-motion` 均已实现，运行时待人工验收）。
- [x] Web 与桌面端不再出现原生 title 双提示（被覆盖按钮已移除 `:title`，仅 More 菜单项等有可见文字的按钮保留）。
- [x] `nuxt typecheck` 通过（EXIT=0）；`app/utils/tooltip-position.test.ts` 6 个用例覆盖 right/bottom 基准定位与视口钳制，全部通过；`workbench-chrome.test.ts` 回归通过。
- [x] 48px Activity Bar 宽度与桌面契约不变（未改布局宽度）。

> 验证边界：`desktop/electron` 子包 typecheck 因该子包 `node_modules` 未安装而失败（`electron` 模块类型缺失），为既有环境问题，与本次改动无关；桌面端运行时观感（悬停手感、8 套主题、减少动态效果）需在桌面端人工验收。

## Out of Scope

- Agent 会话侧栏、文件树行内操作等其他纯图标区（组件已通用，后续可低成本扩展）。
- 「名称 + 一句说明」的富提示文案（用户已选仅名称）。
- 键盘快捷键提示。
- 修改 48px Activity Bar 布局、引入新运行时依赖。

## Decisions（已与用户确认）

- 优化形态：统一 Tooltip 系统（非常显标签侧栏）。
- 覆盖范围：左侧整体（最左图标条 + 工具面板头部）。
- 提示内容：仅按钮名称，沿用现有 i18n 文案。

## 验证发现与修复（2026-08-14）

- **Bug**：小说已打开、IDE 正常渲染时，ActivityBar 的项目按钮长期停留在 disabled 态（opacity 0.34），只有悬停后才恢复可选取观感——用户反馈「先从不可选取状态再变为可选取状态」。
- **根因**：`Tooltip.vue` 最初把「克隆插槽 vnode」写成 `computed`（`renderedSlot`）。computed 只跟踪 `visible` 与插槽函数引用，父组件重渲染产生的新插槽内容（按钮 `disabled`/`class` 变化）不会使其失效，导致克隆出的按钮 DOM 保留旧属性。实测证据：悬停时 Tooltip 文案已是最新（「文件」，无禁用提示），但按钮 DOM `disabled` 仍为 true。
- **修复**：改为普通函数 `renderTrigger()`，在模板中每次渲染调用，基于当前插槽内容重新克隆；并在注释中记录原因防止回归。
- **回归验证**：无头浏览器实测——① 打开小说后全部按钮 enabled；② 回书架后项目按钮 disabled、重开小说后恢复 enabled；③ 切换 Tab 时 active 背景/aria-pressed 实时更新；④ 悬停提示内容、位置、箭头、z-index、`aria-describedby` 均正确。
