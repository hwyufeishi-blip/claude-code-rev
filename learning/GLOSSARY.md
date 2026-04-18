# 术语表

> 按字母序。每条两句话以内。遇到易混术语**必须同时列出易混邻居**。
> 所有条目都附 MAP.md 编号和源码出处，详见 NOTES.md 对应节或 INSIGHTS.md 对应洞察。
>
> 写入约定：
> - 新术语直接 append 到对应字母段
> - 修订释义时保留原位置（不要为了顺序重排）
> - 示例格式见文末模板

---

## A

### AgentTool
承载"主 agent 派生子 agent"的工具。底下有三种派生方式：`runAgent`（新起）/ `forkSubagent`（fork 上下文共享 cache）/ `resumeAgent`（从 sidechain 恢复）。
- **位置**：M3.4 / `src/tools/AgentTool/`
- **易混**：Task（task 是 Agent 执行的容器/生命周期包装，Agent 是里面跑的东西）

### apiMicrocompact
一种上下文压缩策略：在**发 API 请求的最后一刻**做兜底式裁剪。和 microCompact 的差别在触发时机——前者是 API boundary，后者是工具结果落地时。
- **位置**：M5.3 / `src/services/compact/apiMicrocompact.ts`
- **易混**：microCompact（工具结果落地时）、autoCompact（会话级总结）

### Attachment Memory
把 `CLAUDE.md` 和 nested memory 作为 attachment message 注入 prompt 的机制。是 7 种记忆里**唯一直接进 prompt 的**。
- **位置**：M4.7 / `src/utils/attachments.ts`

### autoCompact
到 token 阈值（默认 `contextWindow - 13k`）触发，**整段会话总结**。会话级的最大号压缩。
- **位置**：M5.1 / `src/services/compact/autoCompact.ts`
- **易混**：microCompact（工具结果级）、sessionMemoryCompact（推进 SessionMemory 后清空）

## B

### buildTool
`Tool.ts` 里的工厂函数，吃 `ToolDef` 吐完整 `Tool`。给常用方法塞 fail-closed 默认（`isConcurrencySafe=false` 等）。**所有工具的必经之路**。
- **位置**：M3 / `src/Tool.ts:783`

## C

### checkPermissions
工具接口方法。在 `validateInput` 后、`useCanUseTool` 规则匹配前运行，让工具做**本地前置判断**（比如 Bash 检查当前命令是否属于自己的危险清单）。
- **位置**：M6 判断流水线第 2 步 / `Tool.ts` 接口

### contextCollapse
UI 渲染时把一组 `tool_use` 折叠成一条的展示策略。**不修改消息**，只改渲染。
- **位置**：M5.6 / `src/services/contextCollapse/`
- **易混**：前 5 种 compact 都动消息，这个只动 UI

### Coordinator Mode
一种多 agent 协作模式：一个 agent 当 coordinator 派发子任务，另一个当 worker 执行。
- **位置**：M7 / `src/coordinator/coordinatorMode.ts`

## D

### DreamTask
后台 Task 的一种。空闲时把 daily log 蒸馏成主记忆，对应 M4.6 auto dream。
- **位置**：M7 / `src/tasks/DreamTask/` + `src/services/autoDream/`

## E

### extractMemories
记忆系统的 **turn-end 抽取管道**：每轮结束后台跑一次，扫最近对话，补写 memdir。主 agent 没写到的它捡漏。
- **位置**：M4.3 / `src/services/extractMemories/`

## F

### feature('XXX')
Bun 的 bundle 宏。在**编译期**做死代码消除的条件判断。**必须**写成 `if (feature('XXX'))` 这种直接形态，被提到变量里就不 tree-shake。
- **位置**：M14 / 全项目贯穿
- **关联**：INSIGHTS 看"为什么不用普通 boolean"

### forkSubagent
子 agent 派生方式之一：fork 主线程的上下文和 prompt 字节，共享 prompt cache。比 `runAgent` 省钱且不会因为冷启动差异导致 cache miss。
- **位置**：M7 / `src/tools/AgentTool/forkSubagent.ts`

## G

### GrowthBook
A/B 实验 flag 的运行时源（`feature()` 是编译期源）。用作 `getFeatureValue_CACHED_MAY_BE_STALE` 这种 API。
- **位置**：M14 / `src/services/analytics/growthbook.ts`

## M

### MCP (Model Context Protocol)
Anthropic 的外部工具/资源协议。在本项目里作为**整个模块 M8**存在：传输、认证、连接管理、权限 4 层。
- **位置**：M8 / `src/services/mcp/`
- **易混**：Skill / Plugin / Slash Command —— 这 4 个都是"扩展 agent 能力"的机制但本质不同（见 INSIGHTS）

### memdir
持久记忆目录系统。核心 taxonomy 是 4 种类型（user/feedback/project/reference）；`private` / `team` 是保存与读取时的作用域语义，不是再乘出来的一维类型。相关开关/路径逻辑集中在 `paths.ts`：`isAutoMemoryEnabled()` 是 5 层优先级链，`getAutoMemPath()` 是 3 层解析顺序。
- **位置**：M4.1 / `src/memdir/`
- **易混**：SessionMemory（当前会话一次性摘要，不持久）、Agent Memory（子 agent 专属）

### microCompact
单**工具结果**超阈值时就地裁剪，不总结对话。只对 `COMPACTABLE_TOOLS` 白名单生效（FileRead / Shell / Grep / Glob / WebSearch / WebFetch / FileEdit / FileWrite）。
- **位置**：M5.2 / `src/services/compact/microCompact.ts`

## P

### PermissionMode
整体权限策略，5+1 种：`default` / `plan` / `acceptEdits` / `bypassPermissions` / `dontAsk` / `auto`（ant-only）。
- **位置**：M6 / `src/types/permissions.ts`
- **易混**：PermissionRule（具体规则）、PermissionBehavior（允许/拒绝/询问）

### PermissionRule
具体一条规则，像 `Bash(git *)` 这种。由 `permissionRuleParser.ts` 解析，`shellRuleMatching.ts` 匹配。
- **位置**：M6 / `src/utils/permissions/PermissionRule.ts`

## R

### resumeAgent
子 agent 派生方式之一：从 sidechain 记录恢复之前跑过的 agent。用于后台任务中断后接着跑。
- **位置**：M7 / `src/tools/AgentTool/resumeAgent.ts`

### runAgent
子 agent 派生方式之一：起一个**干净**环境。不共享主线程上下文。
- **位置**：M7 / `src/tools/AgentTool/runAgent.ts`

## S

### SessionMemory
当前会话级的实时摘要 markdown，后台 fork 子 agent 定期更新。
- **位置**：M4.2 / `src/services/SessionMemory/`
- **易混**：memdir（持久 / 跨会话）、Agent Memory（子 agent 专属）

### Skill
给 LLM 看的"技能说明卡"（markdown + frontmatter）。按需加载，通过 `SkillTool` / `DiscoverSkillsTool` 暴露。
- **位置**：M12.1 / `src/skills/`
- **易混**：Plugin（功能包，可含 skill）、MCP（协议）、Command（`/xxx` 本地）

### snipCompact
手动 `/snip` 命令触发的切片压缩。最轻量的 compact，只砍用户指定部分。
- **位置**：M5.5 / `src/services/compact/snipCompact.ts`

### sessionMemoryCompact
把历史对话推进 SessionMemory 后清空原消息。是 autoCompact 的替代路径之一，只在 SessionMemory 模式启用时用。
- **位置**：M5.4 / `src/services/compact/sessionMemoryCompact.ts`

## T

### Task
`src/tasks/*` 下的生命周期容器。8 种：LocalMainSession / LocalShell / LocalAgent / RemoteAgent / InProcessTeammate / LocalWorkflow / MonitorMcp / Dream。
- **位置**：M7 / `src/tasks/types.ts`
- **易混**：Agent（在 Task 里跑的东西）、Tool（Agent 能调的动作）

### Team Memory
memdir 作用域之一。存放整个 team 共享的记忆，通过 `teamMemorySync` 跨成员同步。写入前会过 secret scanner。
- **位置**：M4.1 + M4.5 / `src/services/teamMemorySync/`

### ToolPermissionContext
权限上下文对象。包含当前 mode + 4 个规则桶（allow/deny/ask/stripped）+ 额外 working dirs 等。作为 `DeepImmutable` 传递。
- **位置**：M6 / `Tool.ts`

### ToolUseContext
调用工具时的大对象，携带所有"环境依赖"：options / abortController / readFileState / setAppState / 各种回调 / agentId / messages / 各种 budget state 等。**不用全局单例，显式传入**。
- **位置**：M3 / `Tool.ts:158`
- **关联**：INSIGHTS 看"为什么不用全局单例"

## U

### useCanUseTool
40k 行的大 hook，M6 权限判断流水线第 4 步。负责**规则匹配 + 对话框触发 + 结果回调**。
- **位置**：M6 / `src/hooks/useCanUseTool.tsx`

## Y

### yoloClassifier
`auto` 权限模式下的 LLM 分类器（54k 行）。给待执行工具调用打"安全/危险"分，自动放行安全操作。
- **位置**：M6 / `src/utils/permissions/yoloClassifier.ts`
- **易混**：bashClassifier（只分类 Bash 命令，更小更专）

---

## 模板（新增条目时拷贝）

```md
### 术语名
一句话释义。第二句强调常见误解或使用场景。
- **位置**：Mx.y / `path/to/file.ts`
- **易混**：术语A / 术语B（一句话说清差别）
- **关联**（可选）：INSIGHTS I-00x / NOTES 节
```
