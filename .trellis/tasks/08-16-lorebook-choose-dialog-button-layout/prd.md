# 修复「新建世界书条目」类型选择弹窗布局 + 重设计为卡片式类型选择器

## Goal

修复欢迎页「新建世界书条目」类型选择弹窗的布局问题，并将该弹窗重设计为卡片式类型选择器：类型选项带彩色图标两列展示、取消不再混同于选项按钮（改为标题栏 × / Esc / 遮罩关闭）。

## Background / 已确认事实

- 弹窗由 `app/pages/index.vue` 的 `createWelcomeLorebookEntry` 打开，原实现用共享 `choose` 多动作弹窗（`app/composables/useDialog.ts` 的 `createChooseDialogInstance`）。
- 原布局缺陷 1：420px 弹窗放 6 个按钮（5 类型 + 取消）超宽，flex 压缩按钮导致中文按字符断行成竖排（已实测复现）。
- 原布局缺陷 2：`取消` 与 5 个类型按钮样式完全相同，用户无法区分「选项」与「取消」。
- `getWorkspaceLorebookTypeMeta`（`app/components/novel-ide/workspace/workspace-entry-meta.ts`）提供 5 种类型的图标与彩色样式（location/character/item/rule/note）。
- 共享 `Dialog` 支持 header ×（closable）、Esc/遮罩关闭、隐藏 footer。

## Requirements

- R1 共享 `choose` 弹窗 footer 按钮三种 tone 统一 `px-4` → `px-3` + `whitespace-nowrap`（修竖排，其他调用方受益，已提交 `b5459749`）。
- R2 `useDialog` 新增 `chooseCards`：卡片式多选项弹窗——标题 + header ×（×/Esc/遮罩均解析为 "cancel"）、正文消息 + 两列图标卡片网格、无 footer。
- R3 `index.vue` 的 `createWelcomeLorebookEntry` 改用 `chooseCards`：5 个类型卡片带 `getWorkspaceLorebookTypeMeta` 图标/颜色，取消不再作为选项按钮。
- R4 其他 `choose` 调用方（2–3 按钮确认框）保持共享弹窗不变。

## Acceptance Criteria

- [ ] 打开欢迎页 → 新建世界书条目：弹窗标题「新建世界书条目」，正文提示 + 5 个带彩色图标的类型卡片两列排布；无「取消」按钮混在类型卡片中。
- [ ] 标题栏有 ×，点击 × / 按 Esc / 点击遮罩均关闭并视为取消（不创建文件）。
- [ ] 点击类型卡片后进入路径输入步骤（原流程），创建行为不变。
- [ ] 卡片文字单行、无竖排、无溢出；1280×720 与 900×700 下布局正常。
- [ ] 其他 choose 弹窗（保存/放弃/取消 等）样式与行为不变；共享 footer 按钮仍单行横排。
- [ ] `bun run typecheck` 通过；相关 vitest 无回归。

## Out of Scope

- 不处理英文文案下共享 choose 弹窗 6 按钮单行（世界书入口已改卡片式，不再有此场景）。
- 其他 choose 弹窗不做卡片化。
- 弹窗宽度不调整（卡片弹窗 440px）。

## Key Decisions

- 重设计做成共享 `chooseCards`（卡片式通用能力），但仅世界书入口使用；其他调用方保持原样。
- 取消语义统一走标题栏 ×（含 Esc/遮罩），与类型选项彻底区分。
- 遵循用户 git 策略：直接在本地 master 实现并提交，不创建 PR。
