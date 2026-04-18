# 待解问题

> 格式：`Q编号 · 模块编号 · 一句话问题`
> 解决后**不删除**，只在开头加 `[x]` 并追加答案摘要 + journal 时间戳链接。
>
> 写入约定：
> - 新问题在"未解"区末尾 append
> - 解决时把条目整体移动到"已解"区，前面加 `[x]` 和简短答案 + journal 引用
> - 问题编号单调递增，不复用

---

## 未解

### Q1 · M4.1 memdir · findCanonicalGitRoot 如何判断"canonical"？
当一个仓库有多个 worktree 时，`findCanonicalGitRoot` 需要返回**同一个**根，才能让所有 worktree 共享同一份 auto-memory。具体判断逻辑是啥（读 .git 文件？走 `git rev-parse`？fallback 策略？）还没看。

- **来源**：今日初始化扫 `src/memdir/paths.ts:203` 时遇到
- **出处**：`src/utils/git.ts`（推测）

### Q2 · M4.1 memdir · `isExtractModeActive` 的 GrowthBook 双 flag 怎么互动？
`paths.ts:69` 里 `isExtractModeActive` 先看 `tengu_passport_quail`（总开关），再看 `tengu_slate_thimble`（非交互场景的次级开关）。这两个 flag 具体怎么分工？线上的 rollout 顺序是怎样？

- **来源**：今日 `paths.ts:69` 注释
- **优先级**：低（属于运营细节）

### Q3 · M3 · `buildTool` 为什么用 `TOOL_DEFAULTS` + 展开而不是 class 继承？
这是设计偏好问题，类继承也能达到同样 fail-closed 效果。为什么选对象展开？
- **猜想方向**：保持 tool 是纯 plain object，便于序列化、mock、tree-shake？
- **来源**：`Tool.ts:783`

### Q4 · M7 / AgentTool · "是否走 fork 路径"的真正判断开关在哪？
`renderedSystemPrompt` 分叉（`AgentTool.tsx:496` / `resumeAgent.ts:118`）和 `override.systemPrompt` 汇点（`runAgent.ts:508`）都是 fork 字节的 IO 层，不是"判断是否 fork"的决策层。verifier 报告（journal 02:45 纠错 entry）指出真开关是 `isForkPath` / `isResumedFork` 之类更上游的判断，未深读。
- **来源**：本轮 verifier 报告 P0-#7，journal 02:45 纠错 entry
- **优先级**：中（学到 M7 Agent 线时必查）

### Q5 · 元 · 父模型漂移 → verifier 产出质量漂移（监控项）
verifier.mdc 未指定 `model`，子 agent 继承父 agent 模型。同一条 verifier 规则在不同父模型下产出质量可能不稳定。这是候选设计洞察（**编号未分配**，原"I-005 候选"已被本日 04:02 立下的另一条洞察占用；此候选触发升级时再取下一个可用编号），但暂无证据支持（样本=1，只在 Opus 4.7 下跑过）。
- **触发升级条件**：观察到换父模型后 verifier 质量明显下降 / Cursor 增加 subagent model 白名单
- **监控方式**：切换父模型的新会话，若首次跑 verifier，记下父模型 + verifier 输出质量粗判 → journal `机制理解` 类型 entry
- **来源**：journal 03:01 会话收尾 entry
- **优先级**：低（监控项，不主动查）

---

## 已解

### [x] Q6 · 元 · I-005/I-006 待下次 session 跨 agent 核查
**答（04:08）**：新会话启动时 `.cursor/agents/learning-verifier.md` 已存在 → Task enum 纳入 → 成功调起 learning-verifier，对 I-005/I-006 + 三条 04:02 journal entry 做了完整回扫。判决分布 ✓ 6 / ✗ 1 / ? 3 / ~ 0。✗ 命中 `learning-verifier.mdc` 行数（66 → 实为 68，已就地改 INSIGHTS + 追加 journal 勘误），? 三条分别加 `(未验证)` / `(源码不可验证)` / `(行为自述)` tag。**I-005 注册时机契约由此获实证**（04:02 拒收 vs 04:08 接受，唯一差异变量 = session 启动时文件是否存在）。
- journal 2026-04-17 04:08 entry
- 原问题：本文件历史 Q6 段落（已删）
