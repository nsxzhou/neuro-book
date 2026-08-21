# 修复配置中心 Agent Profile 双栏滚动体验

## Goal

在配置中心的"Agent Profile 模型"页面，左右两栏**各自独立滚动、互不同步**：左侧 Agent Profile 导航列表自己滚（搜索框与默认设置入口保持可见），右侧配置表单自己滚。

## Background

初始需求参考已归档任务 `08-21-fix-provider-list-sticky`（sticky 方案）。第一轮按 sticky 模式对齐后用户反馈仍无效，并澄清真实诉求：左栏列表很长（可超过视口高度），希望左右分别控制滚动。

**诊断结论**（playwright 等价 DOM 复现实证）：

1. sticky 路线对该场景天然失效：列表本身高于可视区时，sticky 无法固定顶部。
2. 真实滚动容器是 `NovelIdeSettingsDialog.vue:890` 的内部 scroller；在其子元素上用 `h-full` 百分比链会退化为内容高度，无法约束布局。
3. 可行方案：面板在 xl 断点 `absolute inset-0` 铺满 section 可视区（包含块 = L888 的 relative section），标题固定，双栏各自 `overflow-y-auto`；aside 作为 grid item 需 `min-h-0` 解除自动最小尺寸，否则行高被内容撑开。

## Requirements

1. xl（>=1280px）：标题区固定；左栏列表独立滚动；右栏表单独立滚动；外层不再整体滚动
2. 小屏（<xl）：保持自然流 + 外层滚动，行为不变
3. 不影响现有功能：Profile 切换、搜索、默认设置入口、状态指示
4. 不提交、不推送代码（用户明确要求）

## Implementation

### 涉及文件

1. `app/components/novel-ide/settings/NovelIdeAgentProfileModelSettingsPanel.vue`
   - 面板根节点：`xl:absolute xl:inset-0 xl:flex xl:flex-col`，标题 `shrink-0`
   - Grid：移除 `min-h-[500px] items-start`，改为 `xl:min-h-0 xl:flex-1`
   - 右栏：`xl:overflow-y-auto`
2. `app/components/novel-ide/settings/AgentProfileNavList.vue`
   - aside：移除 sticky 系列，加 `min-h-0`
   - 列表区：`min-h-[120px]` → `min-h-0 flex-1`

### 验证方式

- `.agent/tmp/sticky-repro-1787281082/` 内等价 DOM 复现页 + playwright 探针：左栏受限内部滚动、右栏独立滚动、外层无整体滚动、互不影响（已通过）
- `bun run typecheck`（nuxt 应用侧）

## Acceptance Criteria

- [ ] 全局/项目范围的 Agent Profile 模型页面：滚动右栏表单时左栏不动，滚动左栏列表时右栏不动
- [ ] Profile 数量多时左栏内部滚动，搜索框与"默认设置"入口始终可见
- [ ] 小屏幕（< 1280px）单列布局不受影响
- [ ] typecheck 通过（desktop/electron 子目录因本地未装依赖报预存错误，与本改动无关）
- [ ] 用户手测确认后归档

## Out of Scope

- 模型设置 Provider 列表页的同等改造（如需要另开任务）
- 提交与推送（用户明确禁止本次自动执行）
