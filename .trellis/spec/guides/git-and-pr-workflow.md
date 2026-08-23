# Git 与 PR 工作流

> 本仓库默认由开发 Agent 在开发者的 **fork** 上工作（`origin` 为上游 `notnotype/neuro-book`，`fork` 为 `nsxzhou/neuro-book`）。本指南定义默认开发方式与 PR 的触发条件，是任务执行与收尾时的行为准则。若与 `AGENTS.md`「Git 工作流」存在冲突，以本指南为准，并向用户说明差异。

## 主线规则：默认直接在 fork 的 master 上开发

1. **默认目标分支是 fork 的 `master`**。每个任务/每次改动，直接在 `master` 上修改、在 `master` 上测试并提交，然后 `git push fork master`。
2. **fork 的 master 领先上游是正常状态**。只要用户没有要求 PR，就保持领先即可，不要因此自动创建分支、自动提 PR、或把改动复制到其它分支。
3. **不为每个任务自动新建分支或 worktree**。除非下面「按需 PR」的触发条件成立，否则一律走 fork master 这个单一主线。

## 分支与 PR：仅在显式请求时创建

4. **只有用户明确发出「提 PR / 提交 PR / 建 PR」等请求时，才创建 PR 分支**。触发条件不成立时，即使是新功能或较大改动，也直接落在 fork 的 master。
5. 创建 PR 分支时，从 **fork 最新 master** 切出，命名遵循 `feat/` / `fix/` / `chore/` 等 Conventional Commit 类型前缀（例：`fix/tooltip-position`），仅用于这一次 PR。
6. PR 完成后，主工作区回到 fork 的 master，不把 PR 分支的改动作为长期主线。

## 准备 PR 之前必读贡献要求

7. 创建或更新任何 PR 前，**必须通读 [`CONTRIBUTING.md`](../../../../CONTRIBUTING.md)**，尤其是「Pull Request 要求」与「Git 与提交」两节，再准备 PR。
8. PR 描述按仓库模板填写，至少包含：
   - 关联的 Issue 编号（轻量改动写「无 / None」）
   - 范围内与明确不在范围内的内容
   - 用户可见结果、技术实现与可能受影响的合同
   - 实际执行的验证命令与结果；未运行的写「未运行」，聚焦测试通过不得写成全量通过
   - 已知限制、后续事项、数据结构/配置/安全/隐私边界是否变化
   - 前端改动的截图、录屏或「未做浏览器验证」的明确说明
9. 一个 PR 只解决一个连贯问题，不夹带无关修复、格式化、依赖升级或文档改动。
10. CI 通过只代表自动检查完成，不代表一定会合并；不要用「CI 过了」代替对 PR 内容的说明。

## 向上游提 PR（区别于 fork 本地按需 PR）

向 `notnotype/neuro-book` 提 PR 时，遵循上游 [`docs/standards/repository-workflow.md`](../../../../docs/standards/repository-workflow.md) 与 `CONTRIBUTING.md`，与 fork 本地开发（fork-master 直开）不同：

- **分支格式必须带引用**：`{type}/{refs}-{slug}`，`refs` 用 `i<issue号>` 或 `t<task号>`（例：`fix/i119-dev-clean-shutdown`）。
- **基线必须是上游最新 master**：从上游最新 `origin/master` 切分支，不从 fork master 切；fork 已合入但上游没有的改动不要夹带进 PR。
- **同步自己的分支用 rebase**：上游 master 前进后 rebase 并自行解决冲突，不 force push、不重写他人提交。
- **PR 前检查 Issue 未被 `status: claimed` 或分配给他人**；完整覆盖用 `Closes #N`，部分覆盖用 `Refs #N`。
- **范围纪律**：一个 PR 只解决一个连贯问题，不夹带格式化、依赖升级、上游合并（upstream merge）或无关修复；上游可能因方向变化/范围过大/无法验证关闭 PR，**可基于更小、更清晰的范围重新提交**。
- **上游行为以测试锁定**：上游对行为改动会补测试断言；PR 若与上游新合同方向相反（如删除上游刻意保留的机制），需先与上游对齐，否则 review 大概率失败。
- **CI 基于旧 base 的绿色不代表可合并**：base 大幅变化（如 monorepo 迁移）后，`mergeable` 会变为 CONFLICTING，必须 rebase 后重跑。

## 合并上游更新（大迁移级）

fork 与上游长期分叉后合并时，采用以下已验证流程（08-23-merge-upstream-monorepo 实证）：

- **干跑预演**：`git merge-tree --write-tree --name-only HEAD <upstream-ref>` 先确认冲突集，不落盘、不碰工作区；结果与预期一致再执行 `git merge --no-ff`。
- **重写型文件必须人工核对**：AGENTS.md / README 等被两边大幅重写的文件，3-way merge 会静默丢失未冲突章节；合并后逐节对照 fork 版本，fork 特有内容（约定、命令、表格）缺失时人工重建。
- **上游行为变化以测试形式显式化**：上游移除行为时会补测试断言（例如 writer preset 只读 Install Root）；合并后先跑冲突相关测试，失败即上游新合同，与 fork 功能冲突时按「保留 fork 功能、去掉与上游合同互斥的部分」裁决。
- **验证对照纯上游基线**：临时 worktree checkout 上游提交跑同一检查（governance / typecheck / 聚焦测试），以「与上游一致、fork 新增 0」为口径；上游自身既有失败不阻塞合并，也不在本任务修复。
- **macOS 测试环境**：`NBOOK_HOST_SYSTEM_TEMP_ROOT` 需指向 realpath（`/var` → `/private/var` 是 symlink，测试根拒绝 symlink）；大小写冲突类测试在大小写不敏感卷上不可跑（模拟 Windows case 语义）。
- **fork 独有目录天然保留**：`.trellis/`、`.pi/`、`.opencode/`、`.githooks/` 等上游不存在的目录 merge 后完整保留，无需特殊处理；`.agents` 等两边共有的目录需确认改动集合无交集。