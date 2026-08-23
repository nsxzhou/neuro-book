# 合并上游 monorepo 迁移 — 执行计划

## 前置检查（开始前）

- [ ] 工作区干净：`git status --porcelain` 为空
- [ ] `git rev-parse HEAD` = `8be31c4f`，`git rev-parse upstream/master` = `197674c3`

## 实施清单

1. **预演确认**：重跑 `git merge-tree --write-tree --name-only HEAD upstream/master`，确认冲突集与预期一致（6 个，无新增）。
2. **执行合并**：`git merge --no-ff upstream/master -m "merge: upstream monorepo migration (t149)"`。
3. **解决 6 个冲突**（按 design.md 映射表）：
   - `.gitignore`、`AGENTS.md`（内容合并，fork 约定优先 + 上游治理指针）
   - `packages/neuro-book/scripts/cli/source-dev.ts`、`writer-writing-reference.ts`、`writer-writing-style.ts`、`shared/source-dev-launcher.test.ts`（逐处合并）
4. **冲突标记检查**：`grep -rn '<<<<<<<\|>>>>>>>' --include='*.ts' --include='*.vue' --include='*.md' --include='*.json' --include='*.tsx' . 2>/dev/null | grep -v node_modules | grep -v '.local'` 为空。
5. **fork 独有内容确认**：合并树包含 `.trellis/`、`.pi/`、`.githooks/`、`.opencode/` 等。
6. **提交 merge commit**（冲突解决后 `git add` 相关文件并完成 merge commit）。

## 验证（每步记录输出）

- [ ] `bun install`（根）成功
- [ ] `bun run governance:check` → `failures=[]`
- [ ] `bun --cwd packages/neuro-book run typecheck` → 无新增失败（对照上游既有失败清单）
- [ ] 聚焦测试：`bun --cwd packages/neuro-book x vitest run server/agent/profiles server/media/image-variant.test.ts shared/source-dev-launcher.test.ts` → 通过
- [ ] `git diff --check` → 干净
- [ ] `git log --oneline -1` 为 merge commit，双亲正确
- [ ] 旧根路径清零：`git ls-tree --name-only HEAD | grep -E '^(app|server|shared|world-engine)$'` 为空

## 检查点 / 门禁

- **门禁 A**（merge 提交前）：冲突全解决、无标记残留、fork 独有内容完整。
- **门禁 B**（push 前）：全部验证通过后，**停下来向用户报告并请求 push 许可**。
- **门禁 C**（push 后）：`git push origin master` 成功，`git rev-list --count HEAD..origin/master` = 0。

## 回滚点

- 合并过程中任一步失败：`git merge --abort` 恢复合并前状态。
- push 前发现问题：`git reset --hard <merge 前 HEAD>`（=8be31c4f）。
- push 后发现问题：`git reset --hard 8be31c4f && git push --force-with-lease origin master`（仅灾难恢复，需用户明确许可）。

## 后续跟进（不在本任务范围）

- 更新 fork 开发文档/命令指引（`bun --cwd packages/neuro-book run dev`）。
- 观察 monorepo 布局下的日常开发流程，必要时给 AGENTS.md 补充操作指引。
