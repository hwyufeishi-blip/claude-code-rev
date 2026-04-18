# 设计洞察 / 反直觉决策

> 这是知识的最高层——**"为什么这样设计，而不那样"**。
> 代码忘了可以再查，设计智慧忘了就没了。
>
> **硬规矩：**
> - 纯 append，**绝不**修改已有条目的 I-00x 编号（编号稳定才能跨文件引用）
> - 写这里的标准：能提出"本可以那样做，但这个项目选了这样做，原因是 X"
> - 只是"机制解释"放 NOTES；只有**可迁移的设计原则**才进这里
> - 每条必须能回答：`我以后做类似设计时，能从这条里拿走什么`

---

## 模板

```md
## I-00X <短标题>
**场景**: 代码里看到的反直觉现象
**反直觉点**: 为什么一眼看去不对
**真正理由**: 它为什么必须这样，或者那样做会出什么问题
**出处**: `src/file.ts:line`
**可迁移经验**: 以后我做什么类型的设计能套用这条
**相关**: I-00x / NOTES 节 / journal 时间戳
```

---

## I-001 append-only 的完整性契约

**场景**: 我们给 `learning/journal/` 定了"纯 append、绝不修改历史"的规矩，但第一次用 agent 初始化 entry 时，时间戳被瞎填（agent 脑补了 `14:00`，实际是 `02:08`）。

**反直觉点**: 从规则合规性看——文件确实是 append-only 的。但日志作为证据系统**已经失效**了，因为时间戳是假的。

**真正理由**: append-only 日志的可靠性是**两半**拼成的：
1. **结构可靠**：历史条目不被修改（文件层面的不变性）
2. **内容可靠**：写入时的元数据（时间戳、引用、数值）必须来自真实测量，不能脑补

只抓第一半，第二半瞎填，日志照样是废纸——更糟，它伪装成可靠来源，会污染下游的交叉引用（INSIGHTS 引用 journal 时间戳、QUESTIONS 标记"已答"并链接 journal、蒸馏扫描最近 N 条）。

推广到整个 agent 行为：**任何"装作知道"比"承认不知道"都危险得多**，因为前者会被下游信任。

**出处**: 
- `.cursor/rules/learning-companion.mdc` 的 "Timestamps (hard rule)" 和 "No fabrication (hard rule)"
- `learning/journal/2026/04/2026-04-17.md` 的 `02:10 纠错` entry

**可迁移经验**:
- 设计任何"不可变日志 / event log / audit trail"系统时，除了不变性约束（append-only、WORM），必须同样强制元数据来自可信时钟源/测量源
- 设计 agent 协议时，把"承认不知道"作为**一等公民的返回类型**，不要允许 agent 用编造兜底。典型对照：HTTP 有 `404` 和 `204`，但有些 agent 只允许"有答案"而不允许"无答案"，就会逼出幻觉
- 对 LLM 来说，温度不等于真实性——看起来合理的填空（`14:00` 而不是 `02:08`）和真的去问时钟是两件事，prompt/rule 必须显式要求后者

**相关**: journal 2026-04-17 02:10 纠错 entry、I-002


## I-002 清单类文档的"完整性语义"必须显式声明

**场景**: 初始化 `PROGRESS.md` 时，L3 记忆线只列了 4 个子模块，但 MAP.md 里其实是 7 个。被用户发现："不是有七种吗？"。检查其他学习线，L1 / L4 / L6 都有同样的静默省略。

**反直觉点**: 从"精简/可读性"角度，挑核心列出来看起来更清爽。但 checklist 类文档**有隐含契约**：读者默认"列出来的就是全部"。静默省略 = 欺骗读者的默认预期。

**真正理由**: 任何清单、路线图、checklist 必须在顶部**显式声明它的完整性语义**：
- "穷尽列表"：列出所有 → 读者可按顺序走完
- "精选子集"：只列核心 → 必须说明"本文件是精选，完整清单看 X"
- "精选 + 标注"：列全部，用 ⭐ 标核心 → 兼顾可读和完整（**推荐**）

这与 I-001 "append-only 的完整性"同源：两者都是**未声明的省略 = 说谎**。一种漏时间戳，一种漏选项，机制完全一样。

**出处**:
- `learning/PROGRESS.md` 的"完整性约定"顶部说明（修复后）
- 修复前问题：L3 记忆线列 4 / 实际 7；L1 工具线列 4 / 实际 8；L4 压缩线列 4 / 实际 7；L6 Agent 线漏 resumeAgent 和 7 种 Task 兄弟

**可迁移经验**:
- 设计任何 schema / enum / 配置 / API 字段列表时，如果是"子集"，必须在注释或文档里明写"这不是全集，全集在 X"
- 代码里的 `switch` / `match` 要么 exhaustive（穷尽所有 case + 编译器校验），要么显式有 `default` 或 "ignore unknown" 分支——不要介于两者之间
- 给 LLM 写 prompt 时也一样：要求它"列出所有 X"和"列出主要 X"是不同任务，prompt 必须选一个并明说

**相关**: I-001（同根问题：未声明的偏差都是失真）、journal 2026-04-17 02:17 纠错 entry、I-003


## I-003 洞察必须回灌到规则，否则不会改变行为

**场景**: I-002 写进 INSIGHTS 后，rule 里没有同步添加"清单完整性"的硬约束。用户第二次发现："你为什么不在 rule 里加上这个呢？"——等于 I-001 立规后的第二次同类失败（第一次是 I-002 本身暴露的"静默省略"）。

**反直觉点**: 写下洞察感觉像"记住了"，但 agent 在下一次类似场景不会自动检索 INSIGHTS。rule 才是运行时加载的上下文。INSIGHTS 没被引用回 rule，就只是**装饰墙纸**。

**真正理由**: agent 的行为层是 rule，不是 INSIGHTS——
- rule 每次对话被 `alwaysApply` 加载，形成行为约束
- INSIGHTS 是参考资料，必须"对话开始时被读到"才可能起作用，而且即使读到也不保证被遵守
- 只写 INSIGHTS 不写 rule = 建好了图书馆但不发书

更深：一个洞察不变成约束，就会**被它自己的复现重新发现**。I-002 就是 I-001 的"静默省略"子集被违反后才抓出来的；如果当初 I-001 立下时就配套了更宽的 "No silent omission" 规则，L1/L3/L4/L6 的漏列就不会发生。

**出处**:
- `.cursor/rules/learning-companion.mdc` 新增 "Insight → rule closure (meta-rule)" 小节
- `.cursor/rules/learning-companion.mdc` 新增 "Completeness semantics (hard rule — I-002)" 小节
- 事件链：I-001（02:11 立下）→ I-002（02:17 因违反泛化） → I-003（02:20 因 I-002 没回灌 rule 而再次暴露）

**可迁移经验**:
- 任何"设计原则 / 事后诸葛 / post-mortem"文档，必须配套更新**强制执行层**（CI 检查、lint 规则、代码评审 checklist、runbook、监控告警），否则原则不会落地
- 对 LLM/agent 系统尤甚：把规则写在 prompt/rule 里 ≠ 把规则写在文档里。前者每次会话生效，后者只是资料
- 建立**洞察 ↔ 规则映射表**作为元约束，让"是否已回灌"这件事本身可见、可审计（rule 里的 `Current insight ↔ rule mapping` 列表就是这个作用）
- 用"规则通过被违反而扩张"的动力学看待规则演进：每次违规是规则边界的信号，不是惩罚对象

**相关**: I-001、I-002（都是被 I-003 描述的循环抓出来的）、journal 2026-04-17 02:20 entry


## I-004 跨 agent 验证的有效性来自 prompt 的物理隔离，不是派生动作本身

**场景**: I-001/002/003 三连环揭示"同一个 agent 既写又检会系统性漏同一个维度"，自然想到"用子 agent 验证"。第一版设计把 verifier 的触发规则和 prompt 都塞进 `learning-companion.mdc`，让父 agent 调 Task tool 派生子 agent 时"从这里抓一段"。用户指出应该独立成 `learning-verifier.mdc`。

**反直觉点**: 只看"派生了子 agent"这一动作，两种方案似乎等价——反正都是调 Task 起一个新 agent。但实际效果天差地别。差异不来自"是否派生"，来自 **prompt 的物理载体**。

**真正理由**: 

子 agent 的"干净"不是 Task tool 给的，是**父 agent 塞给它的 prompt 决定的**：

1. cursor 不会把 workspace 的 mdc 自动注入子 agent；父 agent 必须把 verifier prompt 作为消息显式传入
2. 若 verifier prompt 嵌在父 agent 主流程文件里，父 agent 只有两种选择：
   - 整文件塞 → 子 agent 继承到"怎么写 journal / 怎么更新 NOTES"等动作指令，可能越界去写（污染来自上位文件的权限扩散）
   - 剪切片段 → 易漏、易因上下文裁切把微妙约束丢掉、每次剪切方式还可能漂移
3. 文件独立后，父 agent `Read 整文件 → 原样塞进 Task` 成为**确定性动作**，子 agent 的 context 只包含它该知道的那一段，没有父 agent 历史的回音

更深：干净派生的认知独立性来自"子 agent 看不见父 agent 的历史"，这是信息论意义上的独立性。**派生机制只是必要条件，文件边界才是充分条件**。一个派生到污染 prompt 里的子 agent，本质上是父 agent 的延长线，不是独立判断者。

对应 claude-code-rev 自己的设计选择：`runAgent` vs `forkSubagent` 的分水岭同样不是"是否派生"，而是"派生时继承多少上下文"。verifier 场景要的就是 `runAgent` 的干净；把 prompt 写进共享文件 = 偷偷退化成了 `forkSubagent`。

**出处**:
- 本次对话：用户建议 "单独一个 mdc 比较好" 后的推导
- 待落地：`.cursor/rules/learning-verifier.mdc`（激活契约 + 子 agent prompt 同文件两节）
- `.cursor/rules/learning-companion.mdc` 只保留一行触发指针，指向 verifier 文件
- 类比：`src/tools/AgentTool/runAgent.ts` vs `src/tools/AgentTool/forkSubagent.ts`

**可迁移经验**:
- 设计任何 **reviewer / verifier / judger / critic** 子 agent 时，它们的 prompt **必须住在独立文件**，不能嵌在主流程 prompt 中间。这不是洁癖，是独立性的物理前提
- 一般原则："契约聚合在被调方"（信息专家原则的 agent 系统版本）：谁对事情最有发言权，契约就住在谁家；调用方只保留一行跳转指针
- 推广到多 agent 系统：agent 间的 prompt 依赖关系就是一张图，**独立文件是节点，跨文件引用是边**——可以直接复用软件工程里的模块化 / 依赖反转思维
- 与 I-003 的咬合：I-003 说"洞察要回灌规则"。I-004 补一条——**回灌不仅是"写进去"，还是"写对地方"**。把 verifier 规则塞错文件 = 宣布了规则但让它失效，和没立没区别
- 对 LLM 系统尤其重要：prompt 不像代码有静态分析/类型系统可以抓"变量作用域污染"，边界必须靠**文件本身的物理分割**来强制。架构决策等于边界决策

**相关**: I-001（真实性）、I-002（完整性）、I-003（规则闭环）形成"单 agent 自愈的三件套"；I-004 是**单 agent 自愈触达边界后的外延**——跨 agent 协作的起点。journal 2026-04-17 02:35 entry。


## I-005 custom subagent 的注册时机 = 独立性契约的第二维

**场景**: 把 `learning-verifier` 从 mdc Section B 抽成独立 `.cursor/agents/learning-verifier.md`（I-004 的物理实现），session 中途 `mv` 到位后（此过程时序为本会话行为自述，源码现态不可独立核验，标 `(行为自述)`）立即 `Task(subagent_type: "learning-verifier")` 调用，schema 拒收。拿到反例（用户另一仓库同模式能用）后定位：Cursor 的 Task tool enum 在 session 启动时一次性冻结，中途落位的 agent 文件当前会话不可见。2026-04-17 04:08 新会话实测**反向成立**（启动前文件已在 → 调用被接受），对照实验闭环。

**反直觉点**: 以为"文件在对的位置 + I-004 的独立性契约满足 = agent 可用"。实际还有一条**独立于文件位置**的隐藏契约：注册时机。

**真正理由**: 
I-004 讲"独立文件是物理隔离的充分条件"——这是**结构层**。但 agent 能否被调用取决于"运行时是否把该文件纳入可用 agent 集合"——这是**时机层**。两者正交：
- 结构对（独立文件）+ 时机对（启动前就在）= 可用
- 结构对 + 时机错（session 中途才到）= 当前会话不可用，下次 session 才可用
- 结构错（prompt 嵌在父 rule 里）+ 时机无关 = 永远不独立

同构案例遍地：C 的静态/动态链接（load-time vs dlopen）、Java classpath scan 时机、Python `importlib.reload`、Linux 内核模块 insmod、Kubernetes CRD 创建后 controller 重启——都是"文件/对象存在 ≠ 运行时可见"。

**出处**:
- 本会话 Task 拒收错误（错误字符串为当时自报引用，源码不可独立核验，标 `(源码不可验证)`）：`Error: Invalid arguments: subagent_type: Invalid enum value. Expected 'generalPurpose' | 'explore' | 'shell' | 'best-of-n-runner', received 'learning-verifier'`
- 反例来源：用户另一仓库的 `agents/verifier.md` + `default-verifier-after-plan.mdc` 组合，启动时已存在故能用
- Cursor docs · Subagents · File locations（`.cursor/agents/` 项目级路径）
- **I-005 本身的实证（2026-04-17 04:08 新增）**：Q6 回扫时在"agent 文件 session 启动时已存在"的新会话里，成功通过 `Task(subagent_type: "learning-verifier")` 调起 verifier 并拿到完整 schema 报告——"结构对 + 时机对 = 可用"的正面路径获得直接证据，I-005 的注册时机契约实证闭环（详见 journal 04:08 entry）

**可迁移经验**:
- 设计任何**插件 / 扩展 / 注册式**系统时，把"注册时机（load-time / startup-scan / hot-reload）"显式写进契约文档，不要假设读者会自己推断
- 写"安装指南"时必须包含"装完需要 restart 什么"这一行，不能只写"把文件放到 X 目录"
- 给出安装步骤后，第一个 verification test 应该覆盖"重启后是否可见"，而不是"文件是否在"
- 与 I-004 的咬合：I-004 给出结构契约，I-005 补上时机契约，两者缺一不可达成"独立子 agent 真正可调用"
- 对 LLM agent 设计：任何"父 agent 调子 agent"的协议必须在文档里明写"子 agent 注册发生在何时"，否则父 agent 的 retry 策略会错（它会以为是调用参数错，而不是注册时机错）

**相关**: I-004（本条的前提和结构层搭档）、journal 2026-04-17 04:02 entry、QUESTIONS Q6（本条自述的验证阻塞）


## I-006 规则文件的消费者是 LLM，不是人类读者

**场景**: 第一版 `.cursor/rules/learning-verifier.mdc` 写了 218 行，A/C/D 三节 + 大量 isolation 原理解释 + failure mode 列表 + schema 细节。用户的 `default-verifier-after-plan.mdc` 只有 15 行 4 条 bullet 就完成同等契约职责。用户"按你说的精简吧"一句话把我打回 66 行。

**反直觉点**: 以为规则越详细、边界覆盖越全、原理解释越透 = 规则越 robust。实际相反：rule 越长，父 agent 每次加载成本越高，关键硬约束反而被散文稀释。

**真正理由**:

rule 文件的**唯一消费者是父 LLM agent**，不是人类：
1. `alwaysApply: true` 的 rule 每次对话都被注入上下文，散文和硬约束占用等量注意力 token
2. LLM 注意力是稀缺资源，硬约束（"`✗` 未修完不得宣称完成"）被夹在三段原理解释中间时，命中率下降
3. 人类写技术文档的好习惯（起承转合、背景铺垫、边界讨论、举例）对 LLM 全是噪声——LLM 需要的是祈使句、bullet、条件-动作表
4. 原理解释有它的归宿（INSIGHTS / NOTES），但那是给**人类复盘**或**agent 专门查询设计背景时**读的；每次对话都强制注入的 rule 文件不该承担这个职责

这也是 "为谁写" 的问题：给人写要讲"为什么"，给 agent 写只要讲"下一步做什么"。**把受众当成"会字面执行每一句话的认真新人"** ——前者要你解释缘由避免他误解；后者要你压缩成可执行步骤避免他被解释淹没。

**出处**:
- 改前 `.cursor/rules/learning-verifier.mdc` 218 行（已被覆盖，无法 re-cite 内容；原值为当时自报，未独立核验）
- 改后同路径 **68 行**（`(Get-Content ...).Count` = 68，本次 Q6 回扫 04:08 实测；04:02 原 entry 声称 66，不符，详见 journal 04:08 纠错 entry）
- 参考模板 `default-verifier-after-plan.mdc` 15 行（verifier 核实）
- `.cursor/rules/learning-companion.mdc` 的 I-004 小节现状约 14 行 bullet（verifier 实测 122–135 行；"本轮从 ~50 行缩到 ~15 行 bullet"是过程自述，源码现态无法回溯，标 `(未验证)`）

**可迁移经验**:
- 给 LLM 写 prompt / rule / skill 前，先问"这条文字会改变 agent 的下一步行为吗？"——不会的全删，挪到供复盘/查询的文档层
- rule 层只放：触发条件、硬约束、step-by-step 动作、verdict 处理表。禁止放：原理解释、历史演进、对比论证、failure mode 列表（这些放 INSIGHTS）
- 写完一条 rule 后做一次"替换测试"：把其中某个条件改错或某个约束删掉，agent 是否会触发对应的可观察行为差异？如果删掉都没事 = 这条本来就没起作用 = 应该删
- 给人类写文档的直觉和给 LLM 写 rule 的直觉**系统性相反**，意识到这一点后，两个产出都变好（rule 更犀利，文档更好读）
- 对 skill / subagent prompt 也适用：一个 2000 字的 subagent prompt **不会**比一个 300 字的 prompt 更聪明，只会更慢、更易漂移

**相关**: I-003（洞察 → 规则闭环）——本条是它的"怎么写"层补充；I-004（独立文件的结构契约）；journal 2026-04-17 04:02 entry

