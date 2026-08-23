# 规划本地 PR 分支候选提交上游

## Goal

评估 fork 本地 11 个 PR 候选分支（10 个 Novel IDE UI 修复 + 1 个 writer profile 功能），对照上游现状决定提交策略，产出可执行的 PR 提交计划（含分支、issue、顺序、适配要求）。

## Confirmed Facts（已验证）

- **用户决策**：分组方案 A（5 组左右，见 design.md）；PR 提交按上游规范执行；建上游 issue 纳入规划。
- 12 个本地分支内容早已全部合入 fork master（除 feat/profile-header-layout 有 2 个内容重复、hash 不同的提交外，其余均为 master 祖先）；PR #120（i119）已按用户决定关闭。
- 11 个候选分支的核心改动（排除 .trellis 任务工件）：

| 分支 | 核心改动 | 上游现状 |
|---|---|---|
| feat/profile-header-layout | ProfileTemplateHeader 顶栏图标化 + Tooltip 替换 title | 上游未修 |
| fix/agent-profile-independent-scroll | AgentProfileNavList 列表独立滚动 | 上游未修 |
| fix/bookshelf-navigation | workbench-chrome home 按钮始终显示 | 上游未修（按钮存在）|
| fix/notification-ui-contrast | NotificationViewport 用 theme tokens 提高对比度 | 上游未修（样式不同，需适配）|
| fix/plot-panel-tabs-and-agent-tooltips | AgentChatSurface 头部按钮 Tooltip | 上游未修 |
| fix/profile-workbench-layout | ProfileTemplate 工作台布局与折叠栏 | 上游未修 |
| fix/provider-list-sticky | NovelIdeModelSettingsPanel provider 列表 sticky | 上游未修 |
| fix/tool-panel-remove-duplicate-bookshelf | NovelIdeToolPanel 去重复书架按钮 | 上游未修 |
| fix/tool-panel-upload-button-alignment | Dropdown 上传按钮对齐（1 行）| 上游未修 |
| fix/user-assets-navigation | NovelIdeToolPanel + index.vue 用户资产导航 | 上游未修（userAssetsMode 存在）|
| fix/writer-profile-home-sync | writer-writing-*.ts + novel-workspace.ts 用户自定义文风/参考同步 | 上游有意移除旧 overlay，方向需讨论 |

- 上游 master 无任何对应 UI 修复 issue（搜索 tooltip/notification/sticky/profile/bookshelf 无结果）——按上游 PR 规范需先建 issue。
- 上游 PR 规范要点：分支 `{type}/{refs}-{slug}`、从最新上游 master 切、rebase 同步、一个 PR 一个连贯问题、不夹带上游合并、PR 前确认 issue 未 claimed。

## Requirements

- R1: 对 11 个候选逐一给出提交建议（提/合并/搁置）+ 依据（上游现状、冲突面、价值）。
- R2: 给出 PR 分组与顺序计划（按上游规范，每组一个连贯问题）。
- R3: 识别需要上游讨论或方向对齐的候选（writer-profile-home-sync）。
- R4: 产出每个 PR 的准备清单（issue 前置、分支命名、rebase 要求、适配点）。
- R5: 明确需要用户决策的产品取舍（提交范围、writer 功能方向）。

## Acceptance Criteria

- [ ] 11 个候选各有明确建议（提 / 合并 / 搁置 / 需讨论）及依据。
- [ ] PR 分组与提交顺序明确，符合上游 PR 规范。
- [ ] 每个 PR 的准备清单包含 issue、分支名、适配点。
- [ ] 用户已就提交范围与 writer-profile-home-sync 方向作出决策。
- [ ] 规划产出（prd/design/implement）完整。

## Out of Scope

- 不实际创建 issue / 提交 PR（仅规划；执行需用户另行授权）。
- 不修改产品代码。
- 不处理 PR #120（已关闭）的重新提交，除非用户另行要求。

## Risks / Deferred

- 上游 monorepo 迁移后组件结构可能变化（如 NotificationViewport 样式体系），适配成本未知，需在实施阶段验证。
- 上游可能有自己的体验打磨计划（t150 design language），fork 的修复可能与之重叠或冲突。
- writer-profile-home-sync 涉及产品方向（用户自定义文风/参考），上游是否接受需讨论，可能被拒。
