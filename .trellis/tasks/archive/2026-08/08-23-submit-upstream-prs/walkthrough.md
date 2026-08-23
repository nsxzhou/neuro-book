# 08-23-submit-upstream-prs — walkthrough

## 结果摘要

按规划开始提交上游 PR，因上游 CI 基础设施问题（bun 1.4.0）暂停。PR-1 已完整走通流程（issue → 分支 → 迁移 → 验证 → PR），PR-5 已完成分支迁移待提 PR，全部随暂停归档。

## 已执行

- **PR-1（Profile 工作台）**：
  - issue #124 创建（Agent Profile 工作台 UI 体验修复）
  - 分支 `fix/i124-profile-workbench-ui`（基于上游 master 5a86d066），6 个组件迁移（156+/101-）
  - 适配：VisualEditor 的 `placement="left"` 改为上游支持的 `"bottom"`（上游 TooltipPlacement 输入侧仅 right/bottom）
  - 验证：本地 typecheck ✅ / diff check ✅ / CI Full tests ✅
  - PR #125 创建（Closes #124）→ **因 CI 失败关闭**（见下）
- **PR-5（工具面板与书架导航）**：
  - issue #126 创建 → 已随暂停关闭
  - 分支 `fix/i126-tool-panel-navigation` 完成迁移（5 文件 16+/18-），typecheck ✅ 未提 PR
- **PR-2/3/4**：未开始

## 上游 CI 基础设施问题（暂停原因）

- 现象：PR #125 的 6 个 job（Typecheck/Governance/4×Product）在 `bun install --cwd desktop/electron --frozen-lockfile` 失败：`unsafe folder path "..\..\packages\..."`。
- 根因：CI bun v1.4.0（setup-bun 未 pin）对 desktop/electron 的 `file:../../packages/...` 相对引用（不在 root workspaces）判定 unsafe；本地 bun 1.3.5 同命令成功。同一 run 的 Full tests job（不装 desktop/electron）SUCCESS。
- 影响：任何基于当前上游 master 的新 PR 都会触发；上游 monorepo 迁移后尚无新 PR 对照，无相关 issue。
- 处置：PR #125 留评论说明后关闭；issue #126 关闭；暂停全部 PR 提交，等上游修复（pin bun 版本或调整 desktop/electron 引用）。

## 恢复点

- fork 分支保留：`fix/i124-profile-workbench-ui`（PR-1 完整迁移）、`fix/i126-tool-panel-navigation`（PR-5 完整迁移）均在 origin，可随时重提。
- 上游修复确认方式：上游新 PR 的 CI 转绿 / 出现 bun 版本 pin / desktop 引用调整。
- 恢复时：重开 issue → force-push 分支到新 base（若上游 master 前进）→ 重开 PR。

## 暂停状态

- [x] PR #125 关闭（含 CI 根因评论）
- [x] issue #126 关闭
- [ ] 上游 CI 修复确认后恢复 PR-1/PR-5 → PR-2/3/4
