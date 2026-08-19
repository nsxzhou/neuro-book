# 修复剧本编排面板 Tab 遮挡与 Agent 头部按钮 Tooltip

## Goal

修复两个 UI 问题：
1. 剧本编排面板顶部 Tab 栏（承诺 / 未决 / 章节 / 剧本工作台）在窄视口下内容被遮挡
2. Agent 侧边栏头部按钮添加统一的 Tooltip 效果

## Background

用户截图展示了两个问题：
- 左侧红框：剧本编排面板的 Tab 按钮区域，部分内容被遮挡
- 右侧红框：Agent 侧边栏 header 的按钮缺少 Tooltip 提示

## Issue 1：剧本编排面板 Tab 遮挡

### 位置
`app/components/novel-ide/plot/NovelPlotPanel.vue` lines 1727-1777

### 现状
- Tab 栏容器：`flex shrink-0 items-center justify-between gap-3`
- 左侧标题区：`min-w-0`（可收缩）
- 右侧按钮组：`flex shrink-0 items-center gap-2`
- 每个按钮都用 `shrink-0`，不可收缩
- 4 个按钮：承诺、未决、章节、剧本工作台

### 根因
当视口宽度不足时，右侧按钮组因 `shrink-0` 不会收缩，且容器无 `overflow` 或 `flex-wrap` 处理，导致按钮被容器边界裁剪。

### 修复方案
- 给按钮组容器添加 `flex-wrap` 允许换行，或添加 `overflow-x-auto` 允许横向滚动
- 给按钮添加 `shrink` 允许在必要时收缩
- 确保标题区 `min-w-0` 正确收缩

## Issue 2：Agent 侧边栏头部按钮 Tooltip

### 位置
`app/components/novel-ide/agent/AgentChatSurface.vue` lines 2764-2792

### 现状
头部按钮组包含 7 个按钮：
| # | 功能 | 图标 | title 状态 |
|---|------|------|-----------|
| 1 | 新建对话 | i-lucide-plus | 有 (`:title` + i18n) |
| 2 | 附件面板 | i-lucide-paperclip | 有 (硬编码中文) |
| 3 | 关联 Agent | i-lucide-users | 有 (`:title` + i18n) |
| 4 | 对话树 | i-lucide-git-branch | 有 (`:title` + i18n) |
| 5 | 系统提示词 | i-lucide-terminal-square | 有 (`:title` + i18n) |
| 6 | Session 列表 | i-lucide-messages-square | 有 (`:title` + i18n) |
| 7 | 关闭面板 | i-lucide-x | **无 title** |

所有按钮使用原生 HTML `title` 属性，未使用项目统一的 `Tooltip.vue` 组件。

### 修复方案
- 给关闭按钮（line 2790）添加 `:title` 属性
- 统一升级所有头部按钮，使用项目 `Tooltip.vue` 组件替代原生 `title`
- 附件按钮的硬编码中文改用 i18n
- 保持现有 `title` 文案作为 Tooltip 内容

## Acceptance Criteria

- [ ] 剧本编排面板 Tab 栏在窄视口下不再遮挡按钮内容
- [ ] Agent 侧边栏头部所有按钮均有 Tooltip 提示
- [ ] 关闭按钮有明确的 Tooltip（如"关闭面板"）
- [ ] 附件按钮 Tooltip 使用 i18n
- [ ] 不引入回归：布局正常视口下显示与修复前一致
- [ ] 聚焦测试通过
