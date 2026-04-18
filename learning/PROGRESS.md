# 学习进度

> **最后更新**: 2026-04-17 11:20（落实 I-006：`learning-companion.mdc` 从 230 行精简到 154 行 / 7716 字节，删原理解释/对比论证/failure mode 叙事，保硬约束与 mapping，每轮 `always_applied` 开销 -30%+）
> **当前阶段**: 阶段 1 - 建立全景 (已完成) / 阶段 1.5 - 学习基础设施升级 (已完成) / 阶段 2 - 选择学习线 (待选)
> **学习目标**: 从零到能讲清 claude-code-rev 的整体架构和关键设计思想

---

## 路线图（7 条学习线，映射到 MAP.md 编号）

> **完整性约定**：每条学习线**完整列出**它覆盖的所有 MAP 子模块，不做静默省略。
> - ⭐ 标记的是**核心必看**（吃透这几个就算拿下这条线 80%）
> - 未标 ⭐ 的是**按兴趣挑**（可暂时跳过，但在 checklist 里存在，避免以后误以为"这条线只有 N 个"）
> - 若某一条的价值太低可以明说"L_x 不建议读"而不是静默省略

### 🔧 L1 工具线（推荐起步）
- [ ] ⭐ **M3 契约层**: `src/Tool.ts` - `Tool` 接口 + `buildTool` 工厂 + `ToolUseContext`
- [ ] ⭐ **M3.1 文件工具**: `FileReadTool` 吃透一个最小工具
- [ ] ⭐ **M3.3 执行工具**: `BashTool`（权限、中断、并发）
- [ ] ⭐ **M3.4 Agent 工具**: `AgentTool`（最复杂，留后）
- [ ] **M3.2 搜索工具**: Glob / Grep / ToolSearch
- [ ] **M3.5 计划模式工具**: EnterPlanMode / ExitPlanMode / Brief / ReviewArtifact
- [ ] **M3.6 Web 工具**: WebFetch / WebSearch / WebBrowser
- [ ] **M3.7 MCP 工具**: MCPTool / McpAuth / ListMcpResources / ReadMcpResource（配合 L7 学）
- [ ] **M3.8 元/沟通工具**: AskUserQuestion / SendMessage / Sleep / Skill / DiscoverSkills / TodoWrite / VerifyPlanExecution / Task*

### 🔁 L2 主循环线
- [ ] ⭐ **M2 状态机出口**: `src/query/transitions.ts`
- [ ] ⭐ **M2 核心循环**: `src/query.ts` 主 generator
- [ ] ⭐ **M2 SDK 外壳**: `src/QueryEngine.ts`
- [ ] **M2 Stop 钩子**: `src/query/stopHooks.ts`
- [ ] **M2 配套**: `query/config.ts` / `deps.ts` / `tokenBudget.ts`

### 🧠 L3 记忆线（7 子系统，全部登记）
- [ ] ⭐ **M4.1 memdir 持久记忆**: 结构（4 类型 × 2 作用域）+ 5 层路径解析
- [ ] ⭐ **M4.2 SessionMemory**: 后台 fork 子 agent 的摘要更新机制
- [ ] ⭐ **M4.3 extractMemories**: turn-end 抽取管道，连接 SessionMemory → memdir
- [ ] ⭐ **M4.4 Agent Memory**: 子 agent 专属记忆，3 种作用域（user / project / local）
- [ ] **M4.5 Team Memory Sync**: 跨成员同步 + secret scanner
- [ ] **M4.6 Auto Dream**: 空闲蒸馏 daily log → 主记忆（对应 DreamTask）
- [ ] **M4.7 Attachment Memory**: CLAUDE.md / nested memory 作为 attachment 注入 prompt
- [ ] **程序记忆层（配套）**: `fileStateCache.ts` / `fileHistory.ts` / `promptCacheBreakDetection.ts`

### 🗜️ L4 压缩线（7 种策略，全部登记）
- [ ] ⭐ **M5.1 autoCompact**: 阈值 + 触发 + 整段总结
- [ ] ⭐ **M5 主体**: `compact.ts` 的实现结构（62k，压缩核心）
- [ ] ⭐ **M5.2 microCompact**: 工具结果级裁剪 + `COMPACTABLE_TOOLS` 白名单
- [ ] ⭐ **M5.4 sessionMemoryCompact**: 推进 SessionMemory + 清空模式
- [ ] **M5.3 apiMicrocompact**: 发 API 请求前最后兜底
- [ ] **M5.5 snipCompact**: 手动 `/snip` 切片
- [ ] **M5.6 contextCollapse**: UI 层折叠 tool_use 组（不动消息）
- [ ] **M5.7 postCompactCleanup**: 压缩后清理

### 🛡️ L5 权限线
- [ ] ⭐ **M6 模式枚举**: 5+1 种 PermissionMode（default/plan/acceptEdits/bypassPermissions/dontAsk + auto）
- [ ] ⭐ **M6 规则桶**: `ToolPermissionContext` 的 4 桶（allow/deny/ask/stripped）
- [ ] ⭐ **M6 Tool 契约**: `Tool.checkPermissions` 接口点
- [ ] ⭐ **M6 规则匹配**: `useCanUseTool` 大 hook（40k）
- [ ] ⭐ **M6 主入口**: `permissions.ts` 的 permission 判断总入口（54k）
- [ ] **M6 分类器**: `yoloClassifier`（54k）/ `bashClassifier`
- [ ] **M6 文件系统白名单**: `filesystem.ts`（64k）
- [ ] **M6 规则来源层**: `permissionsLoader.ts` + 5 源覆盖（policy/flag/local/user/project）
- [ ] **M6 高级**: shadowedRuleDetection / denialTracking / bypassPermissionsKillswitch

### 🤖 L6 Agent 线（8 种 Task + 3 种派生）
- [ ] ⭐ **M7 Task 契约**: `Task.ts` + `tasks/types.ts`
- [ ] ⭐ **M7.1 LocalMainSessionTask**: 主会话本身
- [ ] ⭐ **M7 LocalAgentTask**: Task tool 派生的本地子 agent
- [ ] ⭐ **M7 runAgent**: 干净派生
- [ ] ⭐ **M7 forkSubagent**: 上下文共享派生 + prompt cache
- [ ] ⭐ **M7 Coordinator**: 多 agent 协作（coordinator + worker）
- [ ] **M7 resumeAgent**: 从 sidechain 恢复中断的 agent
- [ ] **M7 其他 Task 种类**: LocalShellTask / RemoteAgentTask / InProcessTeammateTask / LocalWorkflowTask / MonitorMcpTask / DreamTask

### 🔌 L7 扩展 / MCP / 桥接（按兴趣挑，**整条线都非核心**）
- [ ] **M8 MCP**: 4 层结构（传输/认证/连接/权限）
- [ ] **M12.1 Skills** vs **M12.2 Plugins** vs **M12.3 Commands** 的边界
- [ ] **M13 远程**: Bridge / Remote / SSH / Teleport 四套并行
- [ ] **M9 Hook 系统**: 9 个 hook 点（配合 L2 学更有感）
- [ ] **M10 UI 层**: Ink + Context 栈 + Vim / Voice 输入
- [ ] **M11 状态层**: 4 份全局状态的分工
- [ ] **M14 横切**: Analytics / Cost / FeatureFlag / Error&Retry（读到时看，不单独学）

---

## 已吃透（勾选前必须在 journal 里有对应条目作为证据）

_待补充_

---

## 本次会话（2026-04-17 02:08 ~ 03:01，学习基础设施升级）

### 学习体系初始化（02:08~02:20）
- ✅ 扫完项目顶层结构，建立 14 大模块坐标系（MAP.md）
- ✅ 搭建 7 文件知识库（MAP/GLOSSARY/INSIGHTS/PROGRESS/NOTES/QUESTIONS/journal + learning-companion rule）
- ✅ 立下 I-001（append-only 完整性）/ I-002（清单完整性语义）/ I-003（洞察必须回灌规则）

### 学习基础设施升级：跨 agent 验证机制落地（02:35~03:01）
- ✅ 立下 I-004（跨 agent 验证的有效性来自 prompt 的物理隔离）
- ✅ 新建 `.cursor/rules/learning-verifier.mdc`（独立 mdc；Section A/B/C/D；generalPurpose + readonly:true；4-verdict schema）
- ✅ 升级 `.cursor/rules/learning-companion.mdc`（加 Learning verification closure 节；carve-out；Default Verifier 关系；schema 兜底；mapping 表补齐 I-001~I-004）
- ✅ verifier 机制首次实战：抓到 I-004 作者（agent）自己把 `runAgent.ts:509` 误定性为 `renderedSystemPrompt` 分叉（实际是 `override?.systemPrompt` 汇点）→ 已发 journal 02:45 纠错 entry
- ✅ 文档↔实战对齐：verifier.mdc A.3 从 `explore` 改为 `generalPurpose + readonly:true` 对齐实战

### E1–E4 回扫已完成（03:07）
- ✅ E1: `MAP.md` M4.1 + `GLOSSARY.md` 的 `memdir` 说明已改正，不再把 type/scope 写成 `4×2=8`
- ✅ E2: `MAP.md` M4.1 已补齐 `memoryAge.ts` / `memoryScan.ts` / `memoryShapeTelemetry.ts`
- ✅ E3: 已在今日 journal 追加 `纠错` entry，澄清 `paths.ts` 里 "5 层优先级" 对应 `isAutoMemoryEnabled()`，而 `getAutoMemPath()` 只有 3 层解析顺序
- ✅ E4: `MAP.md` 的 `src/main.tsx` 已改成 `808857` 字节

### 本次会话未学业务模块
M4.1 memdir 正文（memdir.ts / findRelevantMemories.ts）都还没深读——本次全部精力在**学习基础设施**上而非业务模块。下次回到学习线。

### 本次续会话（2026-04-17 04:02，learning-verifier 重构 + I-005/I-006 立下）
- ✅ 把 learning-verifier 从 `learning-verifier.mdc` Section B 抽出，独立为 `.cursor/agents/learning-verifier.md`（对齐用户仓库 `verifier.md` + `default-verifier-after-plan.mdc` 两文件模式）
- ✅ `learning-verifier.mdc` 从 218 行精简到 **68 行**（04:02 原 entry 声称 66，04:08 实测 68；砍 A/C/D 三节，只留触发 / 跳过 / 调用 / 处理 / carve-out / 分工）
- ✅ `learning-companion.mdc` I-004 小节缩到约 14 行 bullet（实测 122–135 行；"从 ~50 行散文缩来"的过程自述标 `(未验证)`），新增 "Rule writing hygiene (I-006)" + "Agent 文件注册时机 (I-005)" 两段硬规矩
- ✅ 立下 I-005（custom subagent 注册时机契约）+ I-006（rule 消费者是 LLM 不是人类），原"I-005 候选（父模型漂移）"改为未分配编号
- ✅ **[04:08] Q6 跨 agent 核查完成**：I-005 注册时机契约**获跨会话对照实证**（04:02 拒收 + 04:08 接受，唯一差异 = session 启动时文件是否存在）；verifier 10 条 claim → ✓6/✗1/?3/~0；✗ 修完、? 全部加 tag；`learning-verifier.mdc` 行数 66→68 勘误已 journal

---

## 下次优先（按重要性排序）

### P0 · M4.1 memdir 正文深读（原计划的记忆线起步，Q6 既已关闭此为下一自然入口）
- `src/memdir/memdir.ts`（508 行）吃透外部 API（带"它对外暴露哪几个函数？写入 vs 读取怎么分工？"的问题进去）
- `src/memdir/findRelevantMemories.ts`（142 行）一口气读完，重点是检索策略的设计取舍
- 顺手解 Q1（`findCanonicalGitRoot` 如何判断 canonical），在 `src/utils/git.ts`

### P1 · 积压问题池
- Q2（isExtractModeActive 双 flag 互动，优先级低）
- Q3（buildTool 对象展开而非继承的设计原因，L1 工具线时再查）
- Q4（fork 真正判断开关 isForkPath / isResumedFork，M7 时必查）
- Q5（父模型漂移监控，被动项；04:08 补一条粗判：Opus 4.7 续会话 verifier 质量与 02:45 基线持平）

### P2 · 候选洞察
- 候选"父模型漂移→规则产出质量漂移"（未立，编号未分配；触发条件见 Q5；原预占 I-005 已改分配给本日 04:02 立下的"注册时机契约"）
- 候选"自信数值必须双次实测"（未立，样本 n=1——04:02 声称 66 行但 04:08 实测 68 行；下次再遇同类数值失准时升 I-00x）
- "外部提问抓漏" 模式（未立，需 1-2 次复现才升）

---

## 给下次会话 agent 的提示

1. 按 learning-companion rule 开场流程，并行读 MAP / PROGRESS / QUESTIONS / 今日最新 journal（今日 04:08 entry 是最新状态，完成了 Q6 回扫）
2. 本次会话**父 agent 是 Opus 4.7**（续会话）；若下次父模型不同，请在首次跑 verifier 后记下模型名 + 输出质量粗判（对应 Q5 监控）
3. verifier 机制已落地 + 跨会话实证（I-005）+ 会抓数值错（✗ #3 = 本次 66→68 勘误）；**写 learning/ 时必须走**——先跑 verifier 再宣布完成
4. Q6 已关闭，I-005 实证闭环；直接进 **P0 · M4.1 memdir 正文深读**，不用再做基础设施 meta 回环
5. 写入时若出现 `(Get-Content/ls/wc).Count` 类数值声明，写完立刻复跑一次 shell 复核（候选洞察"自信数值双次实测"在跟踪）

---

## 学习体系使用提示

- **开新对话直接说**"继续学"或"我们接着之前的进度"，agent 会读 PROGRESS + 今日 journal 自动续上
- **搞懂一个点时说**"记一下"或"保存"，agent 会沉淀进 journal + NOTES
- **想复盘时说**"整理一下"或"蒸馏"，agent 会把 journal 上浮到 GLOSSARY / INSIGHTS / NOTES
- **发现新问题时**直接问，agent 会自动登记 QUESTIONS
