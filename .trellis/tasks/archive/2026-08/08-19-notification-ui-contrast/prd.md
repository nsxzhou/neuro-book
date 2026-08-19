# 优化通知 UI 视觉对比度

## Goal

将通知 UI 从「半透明玻璃淡色」改为「实心填充」风格，让四类通知（success / warning / error / info）在所有 8 套主题下都清晰可辨，消除"报错也看不见"的问题。

## Background / Confirmed Facts

- 通知组件：[NotificationViewport.vue](file:///Users/zhouzirui/code/neuro-book/app/components/common/NotificationViewport.vue)，通过 `useNotification()` composable 从全局状态渲染。
- 当前视觉问题有 5 层原因：
  1. **背景透明度过低**：`bg-emerald-500/10` 等只有 10-12% 不透明度
  2. **边框透明度过低**：`border-emerald-500/25` 等只有 25% 不透明度
  3. **`backdrop-blur-sm`** 毛玻璃进一步淡化
  4. **硬编码 Tailwind 颜色**：`emerald-500` / `rose-500` / `amber-500` / `sky-500`，不跟随主题
  5. **硬编码白色文字**：`text-white` / `text-white/90`，在所有主题下固定白色
- 项目已有 8 套主题状态变量体系（`theme-tokens.ts`）：
  - `--status-success` / `--status-success-bg` / `--status-success-border`
  - `--status-danger` / `--status-danger-bg` / `--status-danger-border`
  - `--status-warning` / `--status-warning-bg` / `--status-warning-border`
  - `--status-info` / `--status-info-bg` / `--status-info-border`
- 参考文档记录 NotificationViewport 是"跨入口玻璃 toast"的宿主外例外。
- AGENTS.md 要求：颜色一律消费主题变量，不要写 Tailwind 调色板类。

## Requirements

- R1：**实心填充背景**：通知卡片背景使用主题状态色变量 `--status-*` 作为实心填充，不再使用低透明度 + 毛玻璃。
- R2：**主题感知文字**：文字颜色使用 `var(--text-inverse)`（浅色主题下白色、深色主题下暗色文本反色），确保在实心状态色背景上可读。
- R3：**主题感知边框**：边框使用 `var(--status-*-border)` 或 `var(--status-*)` 加深版本，清晰可辨。
- R4：**主题感知圆点**：badge 圆点使用 `var(--status-*)` 变量。
- R5：**四种 tone 可辨识**：success（绿）/ warning（琥珀）/ error（红）/ info（蓝）各自视觉明确。
- R6：**跨主题一致**：8 套主题（light、sepia、dark、catppuccin、dracula、monokai、one-dark-pro、tokyo-night）均表现良好。
- R7：**保留 toast 体验**：圆角、阴影、滑入/滑出动画、自动关闭行为保持不变。

## Acceptance Criteria

- [ ] AC1：错误通知（error）一眼可辨——红色实心背景，白色文字可读，有清晰阴影。
- [ ] AC2：成功、警告、信息通知颜色差异明确，用户不读文字也能区分语义。
- [ ] AC3：移除所有硬编码 Tailwind 调色板类（`emerald-*` / `rose-*` / `amber-*` / `sky-*`），全部改用 `var(--status-*)` 变量。
- [ ] AC4：文字在所有 tone 和所有主题下均清晰可读，无对比度不足问题。
- [ ] AC5：移除 `backdrop-blur-sm`（实心填充风格不需要毛玻璃）。
- [ ] AC6：桌面端 `notification-viewport--desktop` 偏移不受影响。
- [ ] AC7：不修改 `useNotification.ts` composable 的 API 与行为。

## Out of Scope

- 不修改 `useNotification.ts` 的 API、触发逻辑、自动关闭时长。
- 不将 NotificationViewport 移入 `.novel-ide-theme` 宿主。
- 不修改 `notification.error()` / `notification.success()` 等调用点。
- 不新增或修改主题 token（status 变量足够使用）。
- 不修改 Notification 的尺寸、圆角、阴影等整体视觉参数。

## Key Decisions

- **视觉风格**：实心填充风（用户选择）——用主题状态色作为实心背景，白色文字。
- **文字颜色**：使用 `var(--text-inverse)`——浅色主题下白色反色、深色主题下自动适配，保证在所有主题下可读。
- **边框**：使用 `var(--status-*)` 加深版本或同色系边框，替代原有 25% 透明度边框。
- **背景透明度处理**：直接使用 status 变量的实色值，不再叠加 opacity。

## Risks / Mitigations

- **深色主题对比度**：部分深色主题的 status 颜色偏亮（如 `dark` 的 `--status-danger: #f87171`），使用实心背景 + `var(--text-inverse)`（深色主题下为暗色）可能对比度不足。**缓解**：在组件内通过 `color-mix` 加深背景色（如 `color-mix(in srgb, var(--status-danger) 70%, black)`），同时文字使用 `#ffffff` 而非 `var(--text-inverse)`。
- **跨入口兼容性**：NotificationViewport 是宿主外组件，不保证主题变量已加载。**缓解**：使用 `getComputedStyle` fallback 或在 app.vue 的 script 中预加载主题变量。

## Notes

- 单组件优化，集中在 `NotificationViewport.vue`。
- 实现涉及修改：`cardToneClass()` 返回值、`badgeToneClass()` 返回值、模板中的文字颜色与背景样式、移除 `backdrop-blur-sm`。
- 由于 NotificationViewport 在宿主外，需要确保 `--status-*` 变量在全局可用。
