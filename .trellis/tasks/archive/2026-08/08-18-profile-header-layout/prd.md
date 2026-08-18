# Profile 工作台弹窗 header 布局优化

## Goal

解决 Profile 工作台（`UserProfileWorkbenchDialog` → `ProfileTemplateVisualEditor mode="user-profile"`）顶栏的 UI 挤压问题：按钮区过宽、布局拥挤，让顶栏在常用宽度下不再挤压、操作可发现、视觉对齐项目内 World Engine Workbench 顶栏先例。

## Background / Confirmed Facts

- 弹窗链路：`UserProfileWorkbenchDialog.vue`（Dialog size=full）→ `ProfileTemplateVisualEditor.vue`（mode="user-profile"）→ `ProfileTemplateHeader.vue`。
- [ProfileTemplateHeader.vue](../../../app/components/profile-template-editor/ProfileTemplateHeader.vue#L49-L114) 顶栏结构：
  - 左侧：TS logo + `title` / `subtitle`（可截断）。
  - 中部：`FormSelect` 模板选择器，`class="min-w-[320px]"`（占据固定 320px，是宽度大头之一）。
  - 右侧 `ml-auto`：状态文本（`md` 显示）、分隔线（`lg` 显示）、撤销/重做图标、以及 `user-profile` 模式下渲染的**带文字按钮**：预览、编译、编译全部、恢复系统版本、新建、创建 Session、保存（带 `chevron-down` 装饰箭头）、关闭。
- `user-profile` 模式（工作台弹窗）右侧共 9 个控件且多数带长文案，挤压最严重；`system-template` 模式（tsx-profile.preview 页面）右侧只有撤销/重做/预览/验证/保存/关闭，挤压较轻。两模式共用同一 header 组件。
- 项目内先例：World Engine Workbench 顶栏此前做过两次修复——
  - `09ba9073` feat: World Engine Workbench 顶栏图标化：文字按钮改纯图标按钮（`h-8 w-8`）+ `title`/`aria-label`。
  - `60aacb98` fix: 图标按钮统一使用自定义 Tooltip，替代原生 title 慢提示（复用 `app/components/common/Tooltip.vue`）。
- 项目约束（AGENTS.md HTML/Vue）：通用组件优先复用 `app/components/common` 下的 `Tooltip`；Novel IDE 颜色只消费 `app/utils/theme/README.md` 登记的主题变量，不新增调色板。

## Requirements

- R1：`user-profile` 模式下顶栏按钮区不再挤压，常用窗口宽度下所有关键操作可见。
- R2：压缩模板选择器占用宽度（当前固定 `min-w-[320px]`），不挤压标题区与按钮区。
- R3：按钮统一使用 `app/components/common/Tooltip.vue` 提供提示，替代慢速原生 title；颜色仅用已登记主题变量。
- R4：两模式（user-profile / system-template）共用同一 header 组件，样式保持一致（system-template 模式按钮少，自然不挤压）。
- R5：不改后端、不改业务逻辑；`save` / `preview` / `compile` / `create` / `run` 等事件语义保持不变。
- R6（用户决策 Q1=全部图标化）：`user-profile` 模式下右侧全部文字按钮（含保存、创建 Session）统一改为纯图标按钮（`h-8 w-8`），完全复刻 World Engine Workbench 顶栏先例；保存按钮的装饰 chevron 一并移除（当前无下拉行为）。

## Acceptance Criteria

- [ ] AC1：在 1280px 及更宽窗口下打开 Profile 工作台弹窗，顶栏标题区、模板选择器、按钮区均不换行、不溢出、无挤压。
- [ ] AC2：按钮区全部为纯图标按钮（含保存、创建 Session），无文字按钮挤压；每个图标悬浮显示 Tooltip（复用通用 Tooltip 组件）。
- [ ] AC3：保存 / 创建 Session 等主操作以图标形态可见可点，语义与事件不变，保存按钮装饰 chevron 已移除。
- [ ] AC4：模板选择器仍可完整显示所选模板名，宽度不再固定 320px。
- [ ] AC5：`system-template` 模式（tsx-profile.preview 页面）顶栏样式与 user-profile 一致、无回归。

## Out of Scope

- 不改弹窗尺寸、三栏布局与编辑器主体（左组件库 / 中间画布 / 右 Inspector）。
- 不新增保存下拉菜单（当前 save 按钮的 chevron 仅为装饰，无下拉行为，本任务仅移除该装饰）。
- 不引入新主题变量、不改 theme 文档。
- 不做浏览器自动验收（项目约定需用户确认后手动走查）。
- 不改按钮语义、事件名、禁用/加载逻辑。

## 实现记录

### 一、header 布局图标化（主体改动）

- [ProfileTemplateHeader.vue](../../../app/components/profile-template-editor/ProfileTemplateHeader.vue)：右侧带文字按钮（预览 / 编译 / 编译全部 / 恢复系统版本 / 新建 / 创建 Session / 保存）统一改为纯图标按钮（`h-8 w-8`），包上通用 `Tooltip`（placement="bottom"），移除保存按钮的装饰 chevron。
- 模板选择器 `min-w-[320px]` 改为 `min-w-0 max-w-[320px] flex-1`，压缩固定宽度，避免挤压标题区与按钮区。
- 两模式（user-profile / system-template）共用同一 header，样式一致。
- 提交：master `9e934e51`；feat/profile-header-layout `f5f928c6`。

### 二、Tooltip 定位修复

- **问题**：Tooltip 出现位置不在按钮正下方。
- **根因**：处理设备 `.nb-dialog-surface` 带 `transform`，构成新的包含块（containing block）。`ProfileTemplateVisualEditor` 在 Dialog 内部重复挂载嵌套主题宿主，Tooltip 的 `fixed` 浮层被 Teleport 进 `transform` 容器内的嵌套宿主，`fixed` 定位基准失效。
- **修复**：[ProfileTemplateVisualEditor.vue](../../../app/components/profile-template-editor/ProfileTemplateVisualEditor.vue#L2162-L2168) `onMounted` 时先检查是否已处于既有主题宿主内（`closest(.${IDE_THEME_HOST_CLASS})`），仅在不存在时才 `mountThemeHost`，避免嵌套。
- **验证**：用户手动测试确认 Tooltip 已显示在按钮正下方。
- 提交：master `0b621080`；feat/profile-header-layout（cherry-pick）`ff1c8e05`。

### 三、PR 状态（待重新提交）

- feat/profile-header-layout 分支此前提交了 PR，但因上游（origin）存在错误、PR 未过 CI 检测而被撤销。
- 本次 Tooltip 定位修复已同步到 feat 分支并 push fork，**PR 需要重新提交**（待上游错误修复后）——本任务归档时不重新提 PR。

