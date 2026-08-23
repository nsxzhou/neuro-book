# 合并上游 monorepo 迁移到 fork master

## Goal

把上游 `notnotype/neuro-book` 的 monorepo 迁移成果（t149 系列，61 个提交）合并进 fork（`nsxzhou/neuro-book`）的 `master`，同时完整保留 fork 本地 52 个提交的 UI 修复与 fork 特有基础设施（Trellis 工作流、agent 平台配置、pre-push hook）。合并后 fork 继续以 monorepo 布局开发，且不再落后上游。

## Confirmed Facts（已验证）

- 分叉状态：fork `master` = `8be31c4f`（= origin/master），上游 = `197674c3`，merge-base = `5e55c54`；fork 落后 61、领先 52 个提交。
- 上游做了 monorepo 迁移（t149）：主应用迁入 `packages/neuro-book`（app/server/scripts/shared/assets/world-engine/docs 全部移入），六个 sibling 项目收编为 `packages/*`（llmlint、nb-history、nb-memory、nb-workflow、nb-ui、neuro-agent-harness、neuro-book-manager、neuro-book-test-support、file-snapshot-cache、owned-process、neuro-book-contracts），root 只剩 workspace/治理/发布/CI 入口。
- 上游治理变化：开发改为 worktree + PR 分支模式（`{type}/{refs}-{slug}`），新增 `.omp/RULES.md`、`docs/specs/`、`.agents/roles/`、`.agents/tasks/` 体系；**不再有 fork-master 直开模式**。
- 上游新功能：agent asset install runtime（t135）、llmlint evolution lab（t150）、nb-ui 设计语言 token、Agent 取消/对话分支投影等（见 PROJECT-STATUS）。
- 干跑合并（`git merge-tree --write-tree`）：仅 **6 个内容冲突**——`.gitignore`、`AGENTS.md`、`packages/neuro-book/scripts/cli/source-dev.ts`、`packages/neuro-book/server/agent/profiles/writer-writing-reference.ts`、`packages/neuro-book/server/agent/profiles/writer-writing-style.ts`、`packages/neuro-book/shared/source-dev-launcher.test.ts`。git rename 检测把 fork 的 app/server 改动干净映射到新路径；`app/pages/index.vue`、`ProfileTemplateVisualEditor.vue`、`image-variant.test.ts` 等自动合并成功。
- fork 特有目录与上游零重叠：`.agents` 改动集合无交集；`.trellis/`、`.pi/`、`.opencode/`、`.reasonix/`、`.trae/`、`.codex/`、`.cursor/`、`.githooks/` 均为 fork 独有，合并后完整保留。
- 开发入口变化：fork 根 `bun run dev`（旧布局）→ 合并后为 `bun --cwd packages/neuro-book run dev`（上游根 package.json 无 dev script；根 scripts 为治理/发布/CI）。
- 上游已知测试问题（PROJECT-STATUS 记录）：根 typecheck/full tests 存在既有失败（Prisma generated client、隐式 `any`、POSIX `C:/...` fixture，24 files / 78 tests）——合并验证必须聚焦相关范围，不能以全量通过为门槛。
- 6 个冲突文件都很小（112–217 行）。
- 上游已删除 sibling 同步脚本（`sync:nb-history` 等），本地 AGENTS.md 的 sibling 仓库表与同步约定在合并后过时。

## Requirements

- R1: 以一次标准 merge commit 把 `upstream/master` 合入 fork `master`；不 rebase、不 squash、不 force push。
- R2: 解决 6 个内容冲突，保留下述内容：
  - `AGENTS.md`：保留 fork 特有约定（fork-master 直开模式、pre-push hook、Trellis 工作流、Agent 创建 Issue 规则），吸收上游治理入口指针（`.omp/RULES.md`、`docs/specs/`、就近 `AGENTS.md` 路由）；删除已过时的 sibling 同步表；不采用上游 worktree+PR 分支默认（那是 fork 显式决策的差异，见提交 0b52d85a）。
  - `.gitignore`：合并两边忽略规则（上游 global fallback + fork 的 `.githooks` 相关条目）。
  - 4 个代码文件：两边改动合并，行为不回归（fork 的 Ctrl+C 干净停止 dev server、writer profile 改动、测试修正保留）。
- R3: 合并结果通过验证：
  - 根 `bun install` + `bun run governance:check`（failures=[]）；
  - `packages/neuro-book` typecheck（既有失败清单外无新增失败）；
  - 聚焦测试：与冲突文件/本地 UI 修复相关的测试（source-dev-launcher、writer profiles、image-variant 等）；
  - `git diff --check` 干净。
- R4: 合并 commit 落在 fork `master` 并 push 到 fork origin（push 前经用户确认）。
- R5: 保留 fork 独有目录与文件（`.trellis/`、`.pi/` 等 agent 配置、`.githooks/`、novel-style-distill 相关内容），合并树中确认存在。

## Acceptance Criteria

- [ ] `git log --oneline -1` 显示 merge commit，双亲为 fork 旧 HEAD 与 `upstream/master`。
- [ ] 合并树中 `packages/neuro-book/` 存在且为上游版本；`app/`、`server/` 等旧根路径已被上游布局取代（旧入口清零）。
- [ ] 6 个冲突文件全部解决且无冲突标记残留（`grep -r '<<<<<<<'` 为空）。
- [ ] 合并后的 `AGENTS.md` 同时包含 fork 特有约定与上游治理入口，无过时 sibling 同步表。
- [ ] 合并树包含 `.trellis/`、`.pi/`、`.githooks/`（pre-push hook）等 fork 独有内容。
- [ ] 根 `bun install` 成功；`bun run governance:check` 返回 `failures=[]`。
- [ ] `packages/neuro-book` typecheck 通过（除上游记录的既有失败外无新增失败）。
- [ ] 聚焦测试通过（source-dev-launcher、writer profiles、image-variant 相关）。
- [ ] fork 的 pre-push hook 在推送时正常工作（本地分叉已消除，可 fast-forward 语义下推送成功）。
- [ ] fork `origin/master` 已更新到 merge commit（用户确认后）。

## Out of Scope

- 不向 `notnotype/neuro-book` 提 PR 或推送（fork 是私有开发线）。
- 不修复上游既有测试失败（Prisma generated client、隐式 any、POSIX C:/ fixture）——记录并沿用上游口径。
- 不做浏览器人工验收（除非用户另行授权）。
- 不执行发布流程。
- 不迁移 fork 的 Trellis 任务体系到上游 `.agents/tasks` 体系。

## Risks / Deferred

- 合并后的 monorepo 布局会改变 fork 现有开发命令（`bun run dev` → `bun --cwd packages/neuro-book run dev`），相关 AGENTS.md / README 指引随冲突解决一并更新。
- 上游 61 个提交含大量新治理与工具，fork 开发者在 monorepo 布局下需要时间适应；这不是本任务能消化的。
- 合并后 push 属于不可逆性较高的操作；按 R4 在 push 前单独确认。
- 上游测试基线本身有既有失败，验证口径以"无新增失败"为准，不是"全绿"。
