# Directory Structure

> 消费 `@notnotype/owned-process` 的代码位于哪里，以及如何导入。

---

## Overview

本包没有自己的前端目录。消费方分散在仓库各处，统一通过包名导入（根 `package.json` 声明 workspace 依赖 `"@notnotype/owned-process": "workspace:*"`）：

| 消费方 | 文件 | 用途 |
| --- | --- | --- |
| Agent bash 工具 | `server/agent/tools/file-tools.ts` | `runBash()`：bash 命令带 timeout/abort 的完整生命周期 |
| Electron spike | `desktop/spikes/electron/src/main.ts` | `launchProduct()` / `repairProduct()`：启动 Manager Supervisor 并读 NDJSON |
| Source Dev launcher | `scripts/cli/source-dev.ts` | `runSourceDev()`：唯一直接 CLI owner，graceful/forced shutdown |
| Manager | `packages/neuro-book-manager/src/app-commands.ts` | 本机 Product 启动（stdin: "ignore"，Manager 独占宿主 stdin） |

共享契约本体在 `packages/owned-process/src/types.ts` 与 `src/index.ts`，通过 `@notnotype/owned-process` 的 `exports: { ".": "./src/index.ts" }` 暴露（见 `packages/owned-process/package.json`）。

## Directory Layout

```
packages/owned-process/          # 契约与实现（见 backend/directory-structure.md）
└── src/
    ├── index.ts                 # spawnOwnedProcess + 类型 re-export（消费方唯一导入面）
    └── types.ts                 # OwnedProcessSpec / Lease / Completion / TerminationReason / OwnedProcessError
server/agent/tools/
├── file-tools.ts                # runBash 消费
├── file-tools.owned-process.test.ts   # vi.mock 本包的单元测试
└── file-tools.output-cleanup.test.ts  # OutputAccumulator 清理路径测试
desktop/spikes/electron/src/main.ts    # launchProduct / repairProduct 消费
scripts/cli/source-dev.ts              # runSourceDev 消费
shared/source-dev-launcher.test.ts     # runSourceDev 的 lease 测试
packages/neuro-book-manager/src/app-commands.ts  # Manager Product 启动消费
```

## Module Organization

- 消费方把 `spawnOwnedProcess` 当「进程所有权原语」使用，不自己写 `spawn → signal → force kill → stdio close`。
- 一次性命令（版本检查、Git 查询等）仍用普通 `spawn`，不迁入本包（Task 117 D1：只在真实所有权需求暴露时迁入）。
- 消费方各层之间不共享 lease 之外的耦合：AgentJobManager 只通过 reason 请求终止，不接触 Job Object（Task 117 D7 分层）。

## Naming Conventions

- 变量名：`lease`（`OwnedProcessLease`）、`completion`（`Promise<OwnedProcessCompletion>`）、`terminal`（测试中的 deferred promise）。
- 消费方函数：`runBash`、`launchProduct`、`repairProduct`、`runSourceDev` —— 不出现平台名，平台细节由库隐藏。
- 测试 mock：`ownedProcess` / `mocks.spawnOwnedProcess`（见 quality-guidelines.md）。

## Examples

- 最完整的消费组合：`server/agent/tools/file-tools.ts` 的 `runBash()`（timeout + AbortSignal + stdio 转发 + terminationReason 映射）。
- 启动器组合：`desktop/spikes/electron/src/main.ts` 的 `launchProduct()`（NDJSON stdin/stdout 协议 + ready 等待 + shutdown）。
- 类型投影：`scripts/cli/source-dev.ts` 的 `productExit()` 把 `OwnedProcessCompletion` 投影为共享 shutdown 合同。
