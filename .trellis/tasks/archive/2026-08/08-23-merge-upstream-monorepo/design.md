# 合并上游 monorepo 迁移 — 技术设计

## 架构与边界

- **合并方向**：`git merge upstream/master` 到 fork `master`，产生一个 merge commit。不 rebase（重写 52 个 fork 提交，风险高且违反 AGENTS.md 约定），不 squash（丢失 fork 提交历史与上游历史结构）。
- **路径映射**：git 自动 rename 检测把 fork 的旧路径改动（`app/`、`server/`、`scripts/`、`shared/`）映射到 `packages/neuro-book/` 下对应新路径。已用 `git merge-tree --write-tree` 干跑确认仅 6 个内容冲突。
- **fork 独有内容保留机制**：`.trellis/`、`.pi/`、`.opencode/`、`.reasonix/`、`.trae/`、`.codex/`、`.cursor/`、`.githooks/` 等目录上游不存在，merge 天然保留；已验证 `.agents` 改动集合无重叠。

## 冲突解决映射（6 个）

| 文件 | 两边改动内容 | 解决策略 |
|---|---|---|
| `.gitignore` | fork: `.githooks` 相关；上游: global fallback 忽略规则 + zcode 目录 | 合并两边条目 |
| `AGENTS.md` | fork: fork-master 直开 + pre-push + Trellis + sibling 表；上游: monorepo 治理路由 | 以 fork 版为基础，吸收上游治理入口指针（`.omp/RULES.md`、`docs/specs/`、就近 AGENTS.md 路由），删除 sibling 同步表 |
| `scripts/cli/source-dev.ts` → `packages/neuro-book/scripts/cli/source-dev.ts` | fork: Ctrl+C 干净停止 dev server；上游: monorepo 路径适配 | 保留 fork 行为 + 上游路径改动 |
| `server/agent/profiles/writer-writing-reference.ts` | 两边内容改动 | 逐处合并，行为不回归 |
| `server/agent/profiles/writer-writing-style.ts` | 两边内容改动 | 逐处合并，行为不回归 |
| `shared/source-dev-launcher.test.ts` | 两边测试改动 | 逐处合并，测试仍通过 |

冲突文件均 ≤220 行，解决成本低。

## 数据流 / 命令链

- 验证命令：
  - `bun install`（根，workspace 安装）
  - `bun run governance:check`（期望 `failures=[]`）
  - `bun --cwd packages/neuro-book run typecheck`（记录既有失败口径）
  - `bun --cwd packages/neuro-book x vitest run server/agent/profiles server/media/image-variant.test.ts shared/source-dev-launcher.test.ts`（冲突相关聚焦测试）
  - `git diff --check`（无空白错误）
- push：`git push origin master`（fork origin），push 前用户确认。

## 兼容与迁移注意

- 合并后根目录无 `dev` script；开发命令迁移到 `bun --cwd packages/neuro-book run dev`。更新合并后 AGENTS.md 的开发指引。
- 上游已删除 sibling 同步脚本，本地 AGENTS.md sibling 表删除；sibling 仓库内容改为直接从 `packages/*` 获取。
- 上游 git 工作流（worktree + PR 分支）不采用——fork 保持 fork-master 直开（提交 0b52d85a 的显式决策）；冲突解决时不把上游 worktree 规则带进来。

## 回滚

- merge 完成但未 push 前：`git reset --hard HEAD@{1}` 或 `git merge --abort`（未提交时）可完全回退。
- push 后：`git reset --hard <旧HEAD> && git push --force-with-lease origin master`（违反约定，仅灾难恢复用）。
- 每个冲突解决、每轮验证后先 `git commit` 形成检查点，任何一步失败从断点继续。
