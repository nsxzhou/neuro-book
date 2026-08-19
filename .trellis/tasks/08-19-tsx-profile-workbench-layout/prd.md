# 优化 TSX Profile 工作台弹窗模块布局与折叠图标栏

## Goal

解决 TSX Profile 工作台（`ProfileTemplateVisualEditor`，全屏页 `tsx-profile-editor.preview.vue` 与 `UserProfileWorkbenchDialog` 共用）三栏主体区域与折叠组件库图标的三个 UI 问题，让三个模块在常见窗口下都可访问、折叠后的图标列整齐居中、操作可发现。

## Background / Confirmed Facts

- 工作台主体是 [ProfileTemplateVisualEditor.vue](../../../app/components/profile-template-editor/ProfileTemplateVisualEditor.vue#L2239-L2245) 内的三栏 `grid`：
  - 左：组件库（展开 `290px`，折叠 `.component-rail` `42px`）。
  - 中：`ProfileTemplateCanvasPanel`（`minmax(560px,1fr)`，拖拽画布需要宽度）。
  - 右：`ProfileTemplateInspectorPanel`（`minmax(360px,30vw)`，含源码/属性/变量面板）。
- 三栏最小宽度 ≈ 290+560+360+间距≈1250px；外层页面是 `overflow-hidden`（[preview 页面](../../../app/pages/tsx-profile-editor.preview.vue#L17)），窗口窄于该值时右面板被挤出视口且无法横向滚动。
- 折叠组件库图标列样式在 `.component-rail` / `.rail-icon-btn` / `.rail-group-divider`（[rail 样式](../../../app/components/profile-template-editor/ProfileTemplateVisualEditor.vue#L2424-L2477)）；当前存在排列不整齐、未居中问题。
- 折叠态按钮目前用原生 `title`，未复用通用 `Tooltip`（含左侧折叠 `/展开组件库` 按钮、各组件图标按钮，以及右侧 `.panel-rail`）。
- 项目先例：header 已把文字按钮图标化并统一复用通用 `Tooltip`（`app/components/common/Tooltip.vue`, placement="bottom"），见 [ProfileTemplateHeader.vue](../../../app/components/profile-template-editor/ProfileTemplateHeader.vue#L71-L125)；Tooltip 定位 bug 已通过避免嵌套主题宿主修复（前序任务 `0b621080`）。
- 项目约束（AGENTS.md HTML/Vue）：复用 `app/components/common/Tooltip`；颜色只消费已登记主题变量，不新增调色板。

## Requirements（用户已确认的产品取向）

- R1（折叠按钮换 Tooltip，替代 title）：所有折叠态图标按钮——左侧「展开组件库」、各 `library-node-*` 组件图标、右侧「面板」——统一复用通用 `Tooltip` 提供悬浮提示，内容沿用现有 `title` 文案（分组成员/组件名/描述），删除原生 `title`。
- R2（横向滚动兜底，不压缩列宽）：保持三栏默认全展开的现有列宽配置与视觉效果，不改小中画布/右面板的最小宽度（避免挤压画布与面板）；给主体 grid 增加横向滚动，当窗口宽度不足以容纳三栏自然宽度时，靠横向滚动访问最右侧面板。
- R3（折叠图标列对齐居中）：修复折叠组件库图标列的排列不整齐、垂直不对齐、水平不居中问题，让整列图标视觉上统一、对称、可预期（实现时以浏览器实测为准校正像素）。

## Acceptance Criteria

- [ ] AC1：三栏保持默认全展开；窄窗口下主体出现横向滚动，可无遮挡地滚动查看到最右侧 Inspector 面板，画布与面板列宽未被压缩。
- [ ] AC2：折叠组件库显示为整齐的一列图标，各按钮同一高度/宽度、垂直对齐、水平居中、间距统一，视觉不再倾斜或偏移。
- [ ] AC3：折叠组件库的每个图标按钮（含「展开组件库」与各组件项）悬浮显示通用 Tooltip（内容含分组成员/组件名/描述），无原生 title 残留，Tooltip 出现在按钮旁/下方不偏离。
- [ ] AC4：右侧折叠 `.panel-rail` 同样显示 Tooltip；功能与折叠/展开行为无回归。
- [ ] AC5：颜色仅用已登记主题变量；不改后端、不改拖拽/添加节点等业务逻辑与事件语义。

## Out of Scope

- 不压缩三栏显示尺寸、不做三栏拖动调宽或宽度持久化（用户明确倾向不收缩列宽、用横向滚动兜底）。
- 不引入新主题变量、不改 theme 文档。
- 不改 header 布局（前序任务已完成）、不改弹窗尺寸。
- 不做浏览器自动验收（项目约定需用户确认后手动走查）。

## Notes

- 本任务集中在 `ProfileTemplateVisualEditor.vue` 单组件（模板 + 局部样式）；另需 `ProfileTemplateComponentLibraryPanel.vue` / `ProfileTemplateInspectorPanel.vue` 的折叠 trigger 如本身带提示则同步统一。属轻量任务，PRD-only。