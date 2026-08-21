# 优化模型设置 Provider 列表滚动体验

## Goal

在配置中心的"模型设置"页面中，当用户向下滚动填写 Provider 参数时，左侧 Provider 列表保持可见，方便切换不同 Provider。

## Background

当前 `NovelIdeModelSettingsPanel.vue` 使用 CSS Grid 双栏布局，左右两栏共享父级滚动容器。用户滚动右侧配置表单时，左侧 Provider 列表同步滚出视口，切换 Provider 时必须滚回顶部。

**根因**：Grid 布局默认拉伸子元素等高，导致无法为左侧列表设置独立的 sticky 定位。

## Requirements

1. 左侧 Provider 列表在右侧内容滚动时保持可见（sticky 定位）
2. Provider 列表自身内容过多时可独立滚动
3. 不影响现有功能：Provider 切换、新增、删除、启用/禁用
4. 保持响应式：仅在 `xl:` 断点生效双栏，小屏单列不受影响

## Key Decisions

- 采用 CSS `position: sticky` 方案（方案 A），改动最小、无 JS 逻辑
- `top-4`（16px）作为 sticky 偏移量，与父级顶部配置栏保持间距
- Grid 容器添加 `items-start` 防止等高拉伸，使 sticky 正确生效
- 左侧 aside 添加 `h-fit` 使其高度自适应内容而非拉伸

## Implementation

### 涉及文件

- `app/components/novel-ide/settings/NovelIdeModelSettingsPanel.vue`

### 变更清单

1. **L456** Grid 容器：`grid min-h-[500px] gap-5 xl:grid-cols-[260px_minmax(0,1fr)]` → 添加 `items-start`
2. **L458** 左侧 aside：`flex flex-col` → 添加 `sticky top-4 h-fit`

## Acceptance Criteria

- [ ] 全局配置模型设置页面，向下滚动右侧表单时左侧 Provider 列表保持可见
- [ ] 点击左侧 Provider 切换正常工作
- [ ] Provider 列表内容过多时可独立滚动
- [ ] 项目配置范围（`isProjectScope=true`）单列布局不受影响
- [ ] 小屏幕（< 1280px）单列布局不受影响
- [ ] 所有现有功能正常

## Out of Scope

- 不添加滚动阴影效果
- 不修改 Provider 列表视觉样式
- 不重构布局结构
