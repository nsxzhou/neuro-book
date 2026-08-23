# 08-23-merge-upstream-monorepo — walkthrough

## 结果摘要

上游 `notnotype/neuro-book` monorepo 迁移（61 提交）已合入 fork `master`，fork 本地 52 提交与特有基础设施全部保留。合并后 fork 不再落后上游。

## 提交链

```
4bb9889b fix: align merged signal handling and writer preset discovery with tests
3803e403 fix: update task entry references to monorepo layout in owned-process specs
cb2acc9b merge: upstream monorepo migration (t149)   ← merge commit（双亲 8be31c4f + 197674c3）
197674c3 (upstream/master) chore: ignore zcode agent local state directory
```

## 合并执行

- 干跑（`git merge-tree --write-tree`）确认 6 个内容冲突，与预期一致；git rename 检测把 fork 的旧路径改动（`app/`、`server/`、`shared/`）干净映射到 `packages/neuro-book/`。
- 冲突解决：
  - `.gitignore`：两边规则合并（fork `.trellis/workspace/` + 上游包级忽略）。
  - `AGENTS.md`：重构为上游 monorepo 治理为基底 + fork 约定（fork-master 直开、pre-push、Trellis、Agent 创建 Issue 五段式决策）+ 恢复被 3-way merge 误丢的 JS/TS、HTML/Vue、发布流程章节；删除过时 sibling 同步表。注：git 三方合并在重写型 AGENTS.md 上丢失了 fork 多个章节，人工重建。
  - `source-dev.ts` / `source-dev-launcher.test.ts`：信号处理二次信号保持幂等（fork 测试既定意图 + 与上游契约一致），移除合并后闲置的 shutdownNativeProduct import/mock。
  - `writer-writing-style.ts` / `writer-writing-reference.ts`：保留 fork 的用户 `agents/writer/` 自动发现功能，去掉旧 overlay 回退（上游 State Root 架构刻意移除的遗留路径，其测试强制该行为）。
- fork 独有内容保留确认：`.trellis`（121 文件）、`.pi`、`.opencode`、`.trae`、`.githooks/pre-push`、`.agents` trellis skills（46）全部在合并树中。

## 验证结果

| 项 | 结果 |
|---|---|
| `bun install --frozen-lockfile --linker hoisted` | ✅ 94 packages |
| `bun run governance:check` | ⚠️ 1843 failures —— **与纯上游基线完全一致**（上游自身既有 Task ownership hash 漂移），fork 引入 0 新增（修正 2 处 `.trellis/spec` 旧 Task 路径引用后归零） |
| `packages/neuro-book` typecheck | ✅ 通过（需先 `bun run generate` 生成 Prisma client；纯上游同样如此） |
| 聚焦测试（profiles + image-variant + source-dev-launcher） | ✅ 31 files / 257 tests 全过 |
| `git diff --check` | ✅ 干净 |
| 旧根路径清零（app/server/shared/world-engine） | ✅ 迁入 packages/neuro-book |
| 更广 sweep（workspace-files） | ⚠️ 5 失败：4 个 case-collision 测试为 macOS 大小写不敏感文件系统环境性失败（测试模拟 Windows 大小写语义）；1 个 HMR 测试为上游固有模块状态问题。相关文件（project-lifecycle.test.ts、project-session-hmr.test.ts、project-session.ts）与上游逐字节一致且 fork 从未改动——**均非合并引入** |

## 决策记录

1. **merge commit 而非 rebase/squash**：保留双方历史，符合 AGENTS.md 约定（用户批准）。
2. **AGENTS.md 以 fork 约定为基底**：fork-master 直开模式保留（提交 0b52d85a 的显式决策），不采用上游 worktree+PR 模式；上游治理入口指针（`.omp/RULES.md`、`docs/specs/`、就近 AGENTS.md 路由）吸收。
3. **writer preset 去掉旧 overlay 回退**：保留 fork 用户内容功能（`userNbookRoot/agents/writer/`），删除上游 State Root 架构移除的遗留 overlay 读取。可逆，已在报告中说明。
4. **验证口径「与上游基线一致 / 无 fork 新增」而非全绿**：上游自身存在既有失败（governance 1843、workspace-files 环境性失败），不以修复上游问题为范围。

## 待办（门禁 B）

- [ ] 用户确认后 `git push origin master`（push 前先 `git config core.hooksPath .githooks` 确保 pre-push hook 生效）。
- [ ] push 后确认 `git rev-list --count HEAD..origin/master` = 0。
