# Writer Profile Home 同步修复

> 日期：2026-08-21
> 关联：Task 98 (leader.assets 用户资产助手重塑)

## 问题描述

用户在"用户资产"视图编辑了 `workspace/.nbook/agents/writer/` 下的 `styles/` 和 `references/` 文件后，新建小说时这些自定义内容没有被同步到新项目中。

### 现象

- 新建小说后 `agents/writer/` 下没有 `styles/` 和 `references/` 目录
- 即使有，也是系统预设内容，而非用户在"用户资产"视图编辑的内容

### 根因

1. **项目创建时未同步全局 Profile Home**：`copyNovelDirectoryTemplate` 仅合并系统模板和用户模板目录，未处理全局 Profile Home 中的 writer assets
2. **资产加载未读取全局 Profile Home**：`loadWritingStylePresets` 和 `loadWritingReferencePresets` 只从系统预设路径加载，未包含全局 Profile Home 路径

## 修改内容

### 1. 修改 `loadWritingStylePresets` (writer-writing-style.ts)

**文件**: `server/agent/profiles/writer-writing-style.ts`

添加全局 Profile Home 路径作为高优先级候选：

```typescript
const globalStylesRoot = path.join(assetResolver.userRoot, "agents", "writer", "styles");
const roots = candidates ?? (await hasMarkdownFiles(globalStylesRoot)
    ? [globalStylesRoot]
    : [
        path.join(assetResolver.systemRoot, "agent", "profiles", "builtin", "writer.home", "styles"),
        path.join(assetResolver.userRoot, "agent", "profiles", "builtin", "writer.home", "styles"),
    ]);
```

**优先级逻辑**：
1. 如果全局 Profile Home 存在且有用户编辑的内容 → 只用它
2. 否则 → 使用系统预设 + 用户覆盖的预设路径

### 2. 修改 `loadWritingReferencePresets` (writer-writing-reference.ts)

**文件**: `server/agent/profiles/writer-writing-reference.ts`

同样的逻辑应用到 references 加载。

### 3. 修改 `copyNovelDirectoryTemplate` (novel-workspace.ts)

**文件**: `server/workspace-files/novel-workspace.ts`

在项目创建完成后，检查并同步全局 Profile Home 的 writer assets：

```typescript
await syncWriterProfileAssetsFromGlobalHome(projectRoot, userNbookRoot);
```

新增 `syncWriterProfileAssetsFromGlobalHome` 函数：
- 检查全局 Profile Home 的 `styles/` 和 `references/` 目录是否有 `.md` 文件
- 有则复制到新项目的 `agents/writer/` 下

### 4. 新增 `hasMarkdownFiles` 辅助函数

在三个文件中分别添加 `hasMarkdownFiles` 辅助函数，检查目录是否存在且包含 `.md` 文件。

## 行为变化

| 场景 | 行为 |
|------|------|
| 全局 Profile Home **没有** styles/references | 新项目的 `agents/writer/` 下没有这些目录，首次使用 writer agent 时再从系统预设初始化 |
| 全局 Profile Home **有** styles/references | 新项目的 `agents/writer/` 下会立即包含用户自定义的内容 |

## 验证结果

- ✅ `leader-assets-profile.test.ts`：15/15 通过
- ✅ `server/agent/profiles/` 全部测试：226/227 通过（1 个失败是预存的无关问题）
- ✅ `workspace-files.test.ts`：72/72 通过
- ✅ `profile-home.test.ts`：7/7 通过

## 涉及文件

1. `server/agent/profiles/writer-writing-style.ts`
2. `server/agent/profiles/writer-writing-reference.ts`
3. `server/workspace-files/novel-workspace.ts`
