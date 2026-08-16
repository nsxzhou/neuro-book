# 修复「新建世界书条目」类型选择弹窗按钮文字竖排

## Goal

修复欢迎页「新建世界书条目」类型选择弹窗（共享 `choose` 多动作弹窗）中 6 个按钮文字竖排的布局问题，让按钮文字恢复单行横排、布局正常。

## Background / 已确认事实

- 弹窗标题「新建世界书条目」、文案「请选择世界书条目类型」，由 `app/pages/index.vue` 的 `createWelcomeLorebookEntry` 调用 `useDialog().choose(...)` 打开，渲染路径为 `app/composables/useDialog.ts` 的 `createChooseDialogInstance`。
- 弹窗固定宽度 420px，footer 内容可用宽度约 380px；6 个按钮（地点/角色/物品/规则/笔记/取消）自然宽度合计约 398px，超出可用宽度，flex 将每个按钮压缩到 55px，2 字中文按字符断行成竖排（实测 1280×720 下文字为两行：y=389 / y=407）。
- 根因位于共享 choose 弹窗 footer 按钮样式；其他调用方（2–3 按钮）不受影响。

## Requirements

- R1 `createChooseDialogInstance` footer 按钮三种 tone（primary/danger/default）统一 `px-4` → `px-3`，并追加 `whitespace-nowrap`。
- R2 不改变弹窗宽度（保持 420px）、不改 `Dialog.vue`、不改其他调用方。

## Acceptance Criteria

- [ ] 打开欢迎页 → 新建世界书条目 → 类型弹窗：6 个按钮同一 y 基线、单行横排，每个按钮文字单行（无纵向两行、无溢出/截断）；1280×720 与 900×700 两种宽度验证。
- [ ] `bun run typecheck` 通过；既有相关 vitest（world-engine 4 个文件等）无回归。
- [ ] 其他 2–3 按钮的 choose 弹窗无行为变化。

## Out of Scope

- 不处理英文文案下的 6 按钮单行（已知限制，420px 下英文较长文案可能仍放不下）。
- 不做类型选择器卡片式重设计。
- 弹窗宽度不调整。

## Key Decisions

- 修复落在共享 choose 弹窗（根因所在）；其他调用方仅内边距略收紧，无行为变化。
- 遵循用户 git 策略：直接在本地 master 实现并提交，不创建 PR。
