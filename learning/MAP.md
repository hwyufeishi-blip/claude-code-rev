# claude-code-rev 学习地图

> **只读参考**。只有当项目结构发生变化（新增/删除/重命名大模块）时才更新。
> 所有其他 learning/ 文件和 journal 中出现模块时，**统一引用本文件的编号**（例如 M4.1、M5.2）。

---

## 0. 一句话定位

TypeScript + Ink 写的终端 Agent CLI。本质是**"会说话 + 会用工具 + 带 UI 的 AsyncGenerator 状态机"**。主循环是一个不断 yield 消息的异步生成器，工具、记忆、压缩、权限都是挂在这个循环上的子系统。

---

## 1. 顶层依赖图

```
  入口层 (M1) ──启动──▶ 主循环 (M2) ──驱动──▶ 工具系统 (M3)
                           │                     │
                           ├─ 记忆系统 (M4) ◀──读写─┤
                           ├─ 上下文压缩 (M5)      │
                           ├─ 权限系统 (M6) ◀──校验─┤
                           ├─ Agent/Task (M7)     │
                           ├─ MCP (M8)            │
                           └─ Hook 系统 (M9)      │
                           
  UI 层 (M10) ──订阅──▶ 状态层 (M11) ◀──读写── 所有业务模块
  扩展机制 (M12): Skills / Plugins / Commands
  远程/桥接 (M13): Bridge / Remote / SSH
  横切 (M14): Analytics / Cost / FeatureFlag / Error&Retry
```

核心洞察：**业务模块全部围绕 M2 主循环挂载**；UI(M10)/状态(M11) 是独立的另一维。想读懂整体必须先吃透 M2 + M3 + M11 三者的契约。

---

## 2. 模块清单（编号 = 学习坐标）

### M1 入口层
- **一句话**：CLI 参数分发 + 懒加载快速路径（`--version` 零依赖早退）
- **关键入口**：
  - `src/bootstrap-entry.ts`（最外层）
  - `src/entrypoints/cli.tsx`（主 `main()`，按 args[0] 分发，大量 `await import()`）
  - `src/main.tsx`（808857 字节的大入口文件，**别读**）
  - `src/bootstrapMacro.ts`
- **兄弟入口**：`entrypoints/mcp.ts`（MCP server 模式）、`entrypoints/init.ts`、`entrypoints/sdk/`
- **依赖**：几乎所有模块，但全部动态 import

### M2 主循环
- **一句话**：AsyncGenerator 驱动的 agent 状态机
- **核心**：
  - `src/query.ts`（交互式主循环）
  - `src/QueryEngine.ts`（SDK/非交互模式外壳）
- **配套小模块**：
  - `src/query/transitions.ts`（状态机出口类型 Terminal/Continue）
  - `src/query/config.ts` / `deps.ts` / `tokenBudget.ts` / `stopHooks.ts`
- **依赖**：M3 / M4 / M5 / M6 / M9

### M3 工具系统
- **一句话**：统一 Tool 契约 + 50+ 具体工具
- **契约层**：
  - `src/Tool.ts`（接口定义 + `buildTool` 工厂 + `ToolUseContext` 大对象）
  - `src/tools.ts`（注册/聚合）
  - `src/Task.ts`（Task 工具系列的小契约）
- **子类划分**：
  - **M3.1 文件**：FileReadTool / FileWriteTool / FileEditTool / NotebookEditTool
  - **M3.2 搜索**：GlobTool / GrepTool / ToolSearchTool
  - **M3.3 执行**：BashTool / PowerShellTool / REPLTool
  - **M3.4 Agent/Task**：AgentTool / TaskCreateTool / TaskGetTool / TaskListTool / TaskOutputTool / TaskStopTool / TaskUpdateTool / TodoWriteTool / VerifyPlanExecutionTool
  - **M3.5 计划模式**：EnterPlanModeTool / ExitPlanModeTool / BriefTool / ReviewArtifactTool
  - **M3.6 Web**：WebFetchTool / WebSearchTool / WebBrowserTool
  - **M3.7 MCP**：MCPTool / McpAuthTool / ListMcpResourcesTool / ReadMcpResourceTool
  - **M3.8 元/沟通**：AskUserQuestionTool / SendMessageTool / SendUserFileTool / SleepTool / SkillTool / DiscoverSkillsTool / SyntheticOutputTool / MonitorTool / SnipTool / RemoteTriggerTool / TerminalCaptureTool / ScheduleCronTool / WorkflowTool / TungstenTool / OverflowTestTool / EnterWorktreeTool / ExitWorktreeTool / ConfigTool / LSPTool / TeamCreateTool / TeamDeleteTool
- **共享**：`src/tools/shared/` / `src/tools/testing/` / `src/tools/utils.ts`
- **文件结构约定**：每个工具目录 = 本体 + `prompt.ts`（给模型）+ `render*.tsx`（UI）+ `constants.ts`（可选）

### M4 记忆系统（7 子系统并存）
- **M4.1 memdir 持久记忆**
  - 位置：`src/memdir/`
  - 4 种 memory type：`user` / `feedback` / `project` / `reference`；`private` / `team` 是作用域语义，不是再乘出来的一维类型
  - 关键文件：`memdir.ts` / `paths.ts` / `memoryTypes.ts` / `findRelevantMemories.ts` / `memoryAge.ts` / `memoryScan.ts` / `memoryShapeTelemetry.ts` / `teamMemPaths.ts` / `teamMemPrompts.ts`
- **M4.2 SessionMemory（会话级）**
  - 位置：`src/services/SessionMemory/`
  - 后台 fork 子 agent 定期更新当前会话的 markdown 笔记
- **M4.3 extractMemories（抽取管道）**
  - 位置：`src/services/extractMemories/`
  - 连接 SessionMemory → memdir 的泵，turn end 后台抽取
- **M4.4 Agent Memory（子 agent 专属）**
  - 位置：`src/tools/AgentTool/agentMemory.ts` / `agentMemorySnapshot.ts`
  - 3 种作用域：`user`（~/.claude/agent-memory/）/ `project`（.claude/agent-memory/，入 git）/ `local`（.claude/agent-memory-local/，不入 git）
- **M4.5 Team Memory Sync**
  - 位置：`src/services/teamMemorySync/`
  - watcher + 合并；含 `secretScanner.ts` 和 `teamMemSecretGuard.ts`
- **M4.6 Auto Dream**
  - 位置：`src/services/autoDream/` + `src/tasks/DreamTask/`
  - 空闲时把 daily log 蒸馏成主记忆
- **M4.7 Attachment Memory**
  - 位置：`src/utils/attachments.ts`
  - CLAUDE.md / nested memory 作为 attachment 注入 prompt
- **非语义记忆**（程序记忆层）：
  - `src/utils/fileStateCache.ts`（LRU 文件读过没）
  - `src/utils/fileHistory.ts`（文件快照回滚）
  - `src/services/api/promptCacheBreakDetection.ts`（API prompt cache）

### M5 上下文压缩（7 种策略）
- 位置：`src/services/compact/`
- **M5.1 autoCompact**（`autoCompact.ts`）：达阈值整段总结
- **M5.2 microCompact**（`microCompact.ts`）：单工具结果裁剪
- **M5.3 apiMicrocompact**（`apiMicrocompact.ts`）：发请求前最后兜底
- **M5.4 sessionMemoryCompact**（`sessionMemoryCompact.ts`）：历史推进 SessionMemory 后清空
- **M5.5 snipCompact**（`snipCompact.ts` + `snipProjection.ts`）：手动切片
- **M5.6 contextCollapse**（`src/services/contextCollapse/`）：UI 层折叠 tool_use 组
- **M5.7 postCompactCleanup**（`postCompactCleanup.ts`）：压缩后清理
- 主体：`compact.ts`（62k，核心压缩实现）+ `prompt.ts`（压缩用 prompt）
- 钩子：PreCompact / PostCompact / `compactWarningHook.ts`

### M6 权限系统
- **权限模式**（5+1 种）：`src/types/permissions.ts`
  - `default` / `plan` / `acceptEdits` / `bypassPermissions` / `dontAsk`（+ant-only `auto`）
- **规则桶**（4 种，Tool.ts 的 `ToolPermissionContext`）：
  - `alwaysAllowRules` / `alwaysDenyRules` / `alwaysAskRules` / `strippedDangerousRules`
- **判断流水线**：
  1. `validateInput()` — 工具本地参数校验
  2. `checkPermissions()` — 工具自定义前置（Tool.ts 接口）
  3. classifier — `src/utils/permissions/yoloClassifier.ts` / `bashClassifier.ts`（auto 模式）
  4. `useCanUseTool` — `src/hooks/useCanUseTool.tsx`（40k，最重）
  5. PermissionPrompt 对话框
- **规则来源层**（settings 覆盖 5 层，写权限中 projectSettings 被**故意排除**）：
  - `policySettings > flagSettings > localSettings > userSettings > projectSettings`
- **主入口**：`src/utils/permissions/permissions.ts`（54k）+ `permissionSetup.ts`（55k）+ `filesystem.ts`（64k）

### M7 Agent / Task 层级
- **契约**：`src/Task.ts` / `src/tasks/types.ts`
- **8 种 Task**（`src/tasks/` 每目录一种）：
  1. `LocalMainSessionTask`（主会话本身）
  2. `LocalShellTask`（后台 shell）
  3. `LocalAgentTask`（Task tool 派生的本地子 agent）
  4. `RemoteAgentTask`（bridge 远程子 agent）
  5. `InProcessTeammateTask`（进程内 teammate，用户可查看 transcript）
  6. `LocalWorkflowTask`（工作流）
  7. `MonitorMcpTask`（MCP 监听）
  8. `DreamTask`（后台梦境蒸馏，对应 M4.6）
- **子 agent 的 3 种派生**（`src/tools/AgentTool/`）：
  - `runAgent.ts`（新起，干净环境）
  - `forkSubagent.ts`（fork 主线程上下文，共享 prompt cache）
  - `resumeAgent.ts`（从 sidechain 恢复）
- **协调**：`src/coordinator/coordinatorMode.ts`（coordinator + worker 双角色）

### M8 MCP（Model Context Protocol）
- 位置：`src/services/mcp/` + `src/entrypoints/mcp.ts`
- **4 层结构**：
  1. 传输层：`InProcessTransport.ts` / `SdkControlTransport.ts` / stdio / SSE / HTTP
  2. 认证层：`auth.ts`（91k）+ `xaa.ts` / `xaaIdpLogin.ts` + `oauthPort.ts`
  3. 连接管理：`client.ts`（122k）+ `MCPConnectionManager.tsx` + `useManageMCPConnections.ts`
  4. 权限/信任：`channelAllowlist.ts` / `channelPermissions.ts` / `channelNotification.ts` + `elicitationHandler.ts`
- **配置**：`config.ts`（52k）+ `envExpansion.ts` + `headersHelper.ts`
- **SDK shim**：`vscodeSdkMcp.ts`

### M9 Hook 系统
- **9 个 hook 点**：
  - `SessionStart` / `SessionEnd`
  - `PreToolUse` / `PostToolUse`
  - `PreCompact` / `PostCompact`
  - `Stop` / `StopFailure`
  - `PostSampling`
- **入口**：`src/utils/hooks/` + `src/query/stopHooks.ts`
- **来源**：同样 policy/flag/local/user/project 5 层

### M10 UI 层
- 基于 Ink (React for CLI)
- **Context 栈**（`src/context/`，从外到内）：
  - `overlayContext` / `modalContext` / `promptOverlayContext` / `QueuedMessageContext`
  - 侧 context: `voice` / `mailbox` / `notifications` / `stats` / `fpsMetrics`
- **组件**：`src/components/` / `src/screens/` / `src/ink/`
- **REPL 胶水**：`src/replLauncher.tsx` / `src/interactiveHelpers.tsx`（58k）/ `src/dialogLaunchers.tsx`
- **输入模式**：默认 / Vim（`src/vim/` + `useVimInput.ts`）/ Voice（`src/voice/` + `useVoice.ts` 46k）
- **键位**：`src/keybindings/` + `useGlobalKeybindings.tsx`
- **hooks 目录**（REPL 相关）：`src/hooks/`（80+ 个 hook）
- **核心大 hook**：`useReplBridge.tsx`（116k）/ `useTypeahead.tsx`（214k）

### M11 状态层（4 份并存）
- **M11.1 启动期全局单例**：`src/bootstrap/state.ts`（58k）—— sessionId / projectRoot / sdkBetas / turn budget / invoked skills
- **M11.2 REPL 运行时状态**：`src/state/AppState.tsx` + `AppStateStore.ts`（消息、工具进度、UI 状态）
- **M11.3 花费账本**：`src/cost-tracker.ts`
- **M11.4 React Context 栈**：见 M10
- **持久化**：
  - `src/utils/sessionStorage.ts`（transcripts）
  - `src/utils/fileHistory.ts`（文件快照）
  - `src/history.ts`（命令历史）

### M12 扩展机制（3 套并存）
- **M12.1 Skills** `src/skills/`（+ `src/tools/SkillTool/` + `src/tools/DiscoverSkillsTool/`）
  - bundled + loadSkillsDir + mcpSkills
  - 给 LLM 的"技能说明卡"（markdown + frontmatter）
- **M12.2 Plugins** `src/plugins/` + `src/services/plugins/`
  - bundled + builtinPlugins + PluginInstallationManager + pluginOperations
  - 可装可卸的功能包（tools/commands/agents/skills 打包）
- **M12.3 Slash Commands** `src/commands/` + `src/commands.ts`（25k）
  - 用户输入 `/xxx` 触发的本地命令，80+ 个

### M13 远程 / 桥接（3 套并行）
- **Bridge** `src/bridge/` + `src/commands/bridge-kick.ts` —— Claude Code 作为另一台机器的环境
- **Remote** `src/remote/` + `src/commands/remote*/` + `src/moreright/` —— 完整 remote session
- **SSH** `src/ssh/` —— 经 SSH 连远端
- **Teleport** `src/commands/teleport/` + `useTeleportResume.tsx` —— 会话迁移
- **Upstream proxy** `src/upstreamproxy/`

### M14 横切关注点（读到就看，不单独学）
- **Feature Flags**：`feature('XXX')`（bun 宏，编译期 DCE）+ GrowthBook 运行时
- **Analytics**：`src/services/analytics/` + OpenTelemetry
- **Cost / Rate limit**：`cost-tracker.ts` + `services/claudeAiLimits.ts` + `services/rateLimitMessages.ts`
- **Error / Retry**：`src/services/api/errors.ts`（43k） + `withRetry.ts`
- **Token 估算**：`src/services/tokenEstimation.ts` + `src/utils/tokens.ts`
- **Startup profiling**：`src/utils/startupProfiler.ts` + `queryProfiler.ts`
- **LSP**：`src/services/lsp/` + `src/tools/LSPTool/`

---

## 3. 七条推荐学习线（映射到模块编号）

1. **工具线** ⭐ 推荐起步：M3 契约 → M3.1 FileReadTool → M3.3 BashTool → M3.4 AgentTool
2. **主循环线**：M2.transitions → M2.query → M2.QueryEngine → M2.stopHooks
3. **记忆线**：M4.1 memdir → M4.2 SessionMemory → M4.3 extractMemories → M4.4 Agent Memory
4. **压缩线**：M5.1 autoCompact → compact.ts → M5.2 microCompact → M5.4 sessionMemoryCompact
5. **权限线**：M6 mode → Tool.checkPermissions → M6 classifier → useCanUseTool → permissions.ts
6. **Agent 线**：M7 Task → M7 LocalAgentTask → M7 runAgent → M7 forkSubagent → M7 coordinator
7. **扩展/MCP/桥接**（M8/M12/M13）：按兴趣挑

---

## 4. 小白避坑

- 不要读 `src/main.tsx`（808857 字节的大入口文件）
- 看到 `feature('XXX')` 条件分支，默认当 true 读（线上主路径），A/B 分支先跳过
- MCP / GrowthBook / OpenTelemetry / OAuth / LSP 是独立知识域，不懂先跳过，不影响主线
- 碰到 `// Ant-only` / `// Dead code elimination` / `// prompt cache` 注释，**认真读注释比读实现更值钱**
