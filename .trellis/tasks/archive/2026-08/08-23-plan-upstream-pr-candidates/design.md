# 上游 PR 提交规划 — 技术设计

## 分组方案（用户已批准方案 A）

10 个 UI 修复 → **5 个 PR**；writer-profile-home-sync 单独待讨论。

| PR | 主题 | 来源分支（fork 提交） | 涉及文件 | 理由 |
|---|---|---|---|---|
| PR-1 | Profile 工作台 UI 修复 | feat/profile-header-layout + fix/agent-profile-independent-scroll + fix/profile-workbench-layout | ProfileTemplateHeader.vue、ProfileTemplateVisualEditor.vue、ProfileTemplateInspectorPanel.vue、ProfileTemplateLibraryItem.vue、AgentProfileNavList.vue、NovelIdeAgentProfileModelSettingsPanel.vue | 同一批 ProfileTemplate 组件，分开会互相冲突 |
| PR-2 | 通知 UI 对比度 | fix/notification-ui-contrast | NotificationViewport.vue | 独立主题 |
| PR-3 | Agent 头部按钮 Tooltip | fix/plot-panel-tabs-and-agent-tooltips | AgentChatSurface.vue | 独立主题（Tooltip 替换 title）|
| PR-4 | Provider 列表 sticky | fix/provider-list-sticky | NovelIdeModelSettingsPanel.vue | 独立主题 |
| PR-5 | 工具面板与书架导航 | fix/tool-panel-remove-duplicate-bookshelf + fix/user-assets-navigation + fix/bookshelf-navigation + fix/tool-panel-upload-button-alignment | NovelIdeToolPanel.vue、index.vue、workbench-chrome.ts、Dropdown.vue | 同属工具面板/导航区域，NovelIdeToolPanel 被多个改动共享 |
| 待定 | Writer Profile 用户内容同步 | fix/writer-profile-home-sync | writer-writing-*.ts、novel-workspace.ts | 产品方向需上游讨论，默认暂不提交 |

## 每个 PR 的提交流程（上游规范）

1. **建 issue**：上游无对应 issue；每个 PR 先建 1 个上游 issue（描述用户可感知问题 + 现状 + 建议），标 `source: agent`（若上游有该标签约定）。
2. **切分支**：从上游最新 `master` 创建 `fix/i<issue号>-<slug>`（slug ≤5 词 kebab-case）；不 force push，同步用 rebase。
3. **迁移改动**：对应 fork 提交 cherry-pick 到新路径（`app/` → `packages/neuro-book/app/`），解决冲突；适配上游组件现状（如 NotificationViewport 的样式体系差异）。
4. **验证**：`bun --cwd packages/neuro-book run typecheck` + 相关组件测试 + `git diff --check`；浏览器验证按规范标注"未运行"或实测截图。
5. **PR**：模板描述（关联 issue、范围、验证命令结果、未运行项、数据/配置影响）；`Closes #N`。
6. **不自行合并**：交付到 PR 链接，等待上游维护者。

## 顺序与依赖

- 无强依赖；建议按改动独立性从大到小：PR-1 → PR-5 → PR-2/PR-3/PR-4（可并行）。
- 每个 PR 独立评审、独立 merge，互不阻塞；PR-5 内 4 个改动共享 NovelIdeToolPanel/index.vue，必须在同一 PR 内先完成内部合并。

## 适配风险点

- **NotificationViewport**（PR-2）：fork 版用 `useNovelIdeStore` + themeTokens + storeToRefs 集成；上游版本样式体系不同（差异 77 行），需按上游现状重做而非直接 cherry-pick。
- **workbench-chrome.ts**（PR-5）：fork 改动使 home 按钮始终显示（上游有该按钮，2 处）；语义简单，适配风险低。
- **ProfileTemplate 组件**（PR-1）：上游差异行 124/158（Header）、22/573（InspectorPanel）——上游已演进，需逐一对照。
- 所有 UI 组件路径迁移：`app/components/...` → `packages/neuro-book/app/components/...`；`nbook/` 别名的 import 路径需按上游现状调整。

## 回滚 / 可逆性

- 建 issue、提 PR 均可在上游侧关闭/撤回，可逆；PR 被拒不影响 fork master（fork 已合入全部改动）。
- 任何 PR 若上游已自行修复同类问题，放弃该 PR 即可，无成本。
