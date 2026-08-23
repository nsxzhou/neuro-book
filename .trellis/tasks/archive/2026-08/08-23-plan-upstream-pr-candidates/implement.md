# 上游 PR 提交规划 — 执行计划

本任务是**规划任务**：交付完整的分组、顺序、准备清单，供用户批准后另开执行任务。规划阶段不建 issue、不切分支、不改代码。

## 规划产出清单（本任务）

- [x] prd.md：11 候选评估表（Confirmed Facts）+ 需求 + 用户决策（分组 A、按规范、建 issue）
- [x] design.md：5 组 PR 分组、提交流程模板、顺序依赖、适配风险
- [ ] 最终规划摘要呈现给用户并获得批准（本任务收尾）

## 批准后的执行模板（供后续任务使用，不在本任务执行）

每个 PR 的标准步骤（按上游 repository-workflow.md / CONTRIBUTING.md）：

1. **建 issue**（上游仓库）：描述用户可感知问题、现状、建议修复；确认上游无同类 open issue。
2. **切分支**：`git fetch upstream`（或 https://github.com/notnotype/neuro-book.git）最新 master → 创建 `fix/i<issue号>-<slug>`（slug ≤5 词）。
3. **迁移改动**：cherry-pick fork 对应提交到 `packages/neuro-book/` 新路径；解决冲突；按上游现状适配（重点：NotificationViewport 样式体系、ProfileTemplate 组件演进）。
4. **验证**：
   - `bun --cwd packages/neuro-book run typecheck`
   - 相关组件测试（如有）
   - `git diff --check`
   - 浏览器验证：规范要求截图/录屏或标注"未运行浏览器验收"
5. **提 PR**：模板填写（关联 issue、范围内/外、用户可见结果、验证命令与结果、未运行项、数据/配置/安全影响、前端截图）；`Closes #N`。
6. **交付**：报告 PR 链接，不自行合并。

## 分组与建议顺序

| 序 | PR | 来源 | 建议 |
|---|---|---|---|
| 1 | PR-1 Profile 工作台（3 合并） | header-layout + independent-scroll + workbench-layout | 先提（改动集中）|
| 2 | PR-5 工具面板与书架导航（4 合并） | remove-duplicate-bookshelf + user-assets-navigation + bookshelf-navigation + upload-button-alignment | 次之 |
| 3 | PR-2 通知对比度 | notification-ui-contrast | 需重点适配上游样式 |
| 4 | PR-3 Agent Tooltip | plot-panel-tabs-and-agent-tooltips | 独立 |
| 5 | PR-4 Provider sticky | provider-list-sticky | 独立 |
| — | writer-profile-home-sync | — | 待上游方向讨论，默认不提交 |

## 验证口径

- 上游自身 governance:check 有 1843 既有失败（对照过），PR 只需保证不新增；Typecheck/Full tests 是 advisory。
- PR 的 CI 以 GitHub 实际结果为准；`mergeable` 需在 base 前进后确认，冲突则 rebase。

## 回滚点

- 规划阶段无代码改动，无回滚需求。
- 执行阶段：issue/PR 均可关闭撤回；PR 被拒不影响 fork master。
