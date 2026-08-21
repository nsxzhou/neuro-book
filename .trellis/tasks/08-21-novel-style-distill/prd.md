# 新增 novel-style-distill 文风蒸馏 skill

## Goal

用户在 Agent 会话中引用 writer 的参考资料（references）后，Agent 能按固定方法论蒸馏出 writer 可直接选用的文风预设（styles），全程纯 skill 指令实现，不改任何服务端代码。

## Background（已验证事实）

- Writer 的 styles/references 存在三层：系统内置种子 `assets/workspace/.nbook/agent/profiles/builtin/writer.home/{styles,references}`；全局用户级 `<运行时 Workspace Root>/.nbook/agents/writer/{styles,references}`；项目级 `<project>/agents/writer/{styles,references}`。
- 读取顺序为项目优先 → 全局兜底（`createLayeredProfileHomeFacade`，`server/agent/profiles/profile-home.ts:82`）；服务端 UI 资源写操作只落项目级。
- 会话形态决定目标层：项目内会话 cwd=项目根（`authorized-file-operation.ts:78`），用户资产页会话（`WorkspaceKind: "user-assets"`，`app/stores/novel-ide.ts:142`）cwd=运行时 Workspace Root 且 writer home 只有全局层（`server/agent/profiles/profile-http-service.ts:133-146`）。
- 生成物合同：style frontmatter 七字段 `key/label/sourcePreset/identifier/name/enabled/role`（`server/agent/profiles/writer-writing-style.ts:10`）；reference frontmatter `key/label/sourceTitle/sourceChapters/generatedFrom`（`server/agent/profiles/writer-writing-reference.ts:11`）；项目级 home 读取用 partial 校验，裸文件可用。
- 现成范本：`writer.home/references/reborn-villain-loli-magic-girl.first-three-chapters.md` → 同名 `.style.md`（`<Writing_style>` 包裹 + 约 15 个分析维度）。
- Skill 发现机制：目录含 `SKILL.md` 即被收录，frontmatter 解析 `name/description/when_to_use`（`server/agent/skills/skill-catalog.ts`）；系统级 skill 放 `assets/workspace/.nbook/agent/skills/<name>/`。

## Requirements

1. 新建 `assets/workspace/.nbook/agent/skills/novel-style-distill/SKILL.md`，frontmatter 含 `name/description/when_to_use`。
2. Skill 指令覆盖完整流程（熔铸法，方法论取自 `.agent/refs/casting-workflow`，MIT）：
   - 目标层判定：cwd 下有项目标志（如 `manuscript/`）→ 项目路径 `agents/writer/`；否则 → 全局路径 `.nbook/agents/writer/`。
   - 引用驱动：处理用户引用的 reference 文件；未引用时列出当前层 references 目录供选择。
   - 裸文件（无 frontmatter）在生成过程中自动派生元数据，不写回 reference 原文。
   - 指纹提取：逐篇实测表层节奏（！/句、，/句、对话密度、段落长度）、句式、词汇、结构四类指纹，禁止编造数字。
   - 多篇引用 → 融合成一个混合 style，按机械化互消规则判定：过半篇共有 = 类型公约数保留；仅 1 篇独有 = 作者指纹剔除；其余存疑交用户裁决。风格级元素学公约数，内容级元素一律改写自创，「宁可远不可近」。
   - 单篇引用退化为显式指纹剥离：可迁移技法层进 style，作者专属层全部进 [核心禁区]。
   - 分析框架沿用现有样例维度结构并新增 [量化节奏指标] 维度（实测数字换算为写作目标值），允许按风格特征增删专属维度。
   - 反 AI 味基线默认写入 [核心禁区]（禁用模板描写、禁用句式、思维标记限频），与 llmlint 口径一致；源文本身依赖的项标注「源文特征」例外。
3. 产出 frontmatter 严格对齐 style 合同：`identifier` 用 uuid、`enabled: false`、`role: system`、`name` 沿用 `📝文风（选一）|<label>` 格式、`sourcePreset` 指回来源 reference 的 label（多篇时全部列出）。
4. 边界：不修改 references 下任何文件；不自动启用（不切 writingStylePreset）；不动系统内置种子层。

## Acceptance Criteria

- [ ] 测试 Project 的 `agents/writer/references/` 放一篇带 frontmatter + 一篇裸 md，分别引用后均能生成合法 style 文件（七字段齐全、类型正确）。
- [ ] 同时引用两篇 → 只产出 1 个融合 style，`sourcePreset` 覆盖两个来源。
- [ ] 项目会话生成的 style 出现在该 Project writer 设置的文风下拉（project-scope 资源发现）。
- [ ] 用户资产工作区会话生成的文件落在 `.nbook/agents/writer/styles/`，且任意项目的文风下拉可见（origin global）。
- [ ] `references/` 下文件零改动；生成的例文与源文无长串重复。
- [ ] skill 被 catalog 正常发现（name/description/when_to_use 可解析）。

## Out of Scope

- 全局 writer.home 系统内置种子层编辑；批量全目录生成；workflow 编排化；Casting-Workflow 其余能力（指纹蒸馏等）；项目模板加骨架目录；服务端代码改动。

## Risks / Deferred

- 融合质量依赖模型发挥，skill 只能锁结构与纪律；后续可用 write-review-loop 类 workflow 加审校环。
- 全局层产物从项目 UI 不可编辑/删除（既有产品行为，非本任务引入）。
- skill 命名可再改，成本低。

## Notes

- 轻量任务，PRD-only。
- 实现时应同步把新 skill 登记进 `novel-guide/SKILL.md` 的「随时可用层」表格，保持路线图准确。
