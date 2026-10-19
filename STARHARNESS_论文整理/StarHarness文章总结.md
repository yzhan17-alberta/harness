# StarHarness 文章总结

> 论文：*StarHarness: Evolving Harnesses with Stratified Search for Enterprise Environments*（Esakkivel Esakkiraja, Denis Akhiyarov, Vikas Yadav, Sai Rajeswar, Patrice Bechard, Sridhar Nemala, Sagar Davasam；ServiceNow / Mila / Université de Montréal；arXiv:2608.24804，2026 年 8 月）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：在模型权重固定的前提下，能否通过搜索演化出环境专属的 agent harness，来消除工具密集型企业任务中持续存在的"模型–环境不匹配"。 | Abstract；§1 |
| 子问题：（1）能否从紧凑的任务子集演化 harness 而不过拟合该子集；（2）演化出的改动能否迁移到未见任务和其他模型；（3）harness 演化修复了哪些交互失败。 | §1 |
| 形式化目标：h* = argmax_{h∈H} J(h, D_holdout)，J 为模型 M 用 harness h 在数据集上的平均任务分；搜索时不能访问 holdout 结果，用 proposer 可见的 search 集与 proposer 不可见的 selection 集近似。 | §3.1 式 (1) |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被优化对象：Stirrup agent harness 的可编辑面——prompt 与任务框架、工具定义与 schema、参数预处理、skills、MCP providers、subagent 结构、上下文管理、验证、finish 逻辑。 | §3.1 |
| 方法本身：StarHarness——按 baseline 失败行为分层抽样构造评测池；拆分 proposer 可见 search 集、proposer 不可见 selection 集和 holdout 集；proposer 提出 git-diff 补丁，经 validator、test flip、selection 评估和确定性接受规则后提交；支持爬山与树搜索两种搜索过程；由基于 Oh My Pi 的 coding harness 执行整个循环。 | §3.1–§3.3；Fig. 1 |
| 任务领域：企业 IT 运维与财务流程——Kubernetes 根因分析（ITBench SRE）、ITSM 状态变更流程（EnterpriseOps-Gym）、多应用财务流程（AutomationBench Finance）。 | §2.2；§4.1 |
| 底层模型：演化时 agent 为 GPT-5.4（medium reasoning）；冻结迁移评估 GPT-5.4-mini (medium)、GPT-5.5 (medium；EnterpriseOps 为 high)、Qwen3.5-27B（ITBench）、Qwen3.6-27B（EnterpriseOps、AutomationBench）。 | §4.1；Table 1 |
| 优化器 / 判官 / 专家模型：proposer 为 GPT-5.4，运行在 Oh My Pi（Pi 的变体）coding harness 内；是否使用 LLM 判官未说明，分数按各基准验证器与 Artificial Analysis 评估描述给出。 | §3.1；§4.1 |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| SRE 根因分析 | ITBench-AA SRE 最新公开版，40 个 Kubernetes 场景（OpenTelemetry demo 应用），离线事故快照含 alerts、events、traces、metrics、topology；输出结构化 JSON 诊断。 | §2.2；§4.1 |
| ITSM 流程 | EnterpriseOps-Gym：103 个 ITSM oracle 任务（incident、problem、change、knowledge、user management），取自 1,150 题全集；ServiceNow MCP 后端；SQL 验证器检查最终数据库状态。 | §2.2；§4.1 |
| 财务流程 | AutomationBench Finance：100 个任务（AP/AR、expenses、reporting、bookkeeping），47 个模拟 SaaS 应用；程序化断言判分；guardrail 违规记 0；作者的 Finance-100 子集与 Stirrup harness 与原基准论文默认设置不同。 | §2.2；§4.1 |
| 切分 | 从 N 个任务保留 N′ 个可复现评估任务；抽 K≈N/2 为评测池（按 baseline 失败模式、baseline 分数、verifier 通过率分层）；再拆为 search / selection（匹配三项分布）；其余 N′−K 为 holdout。各基准 K 的具体数值：未说明。去污染：未说明。 | §3.2 |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | Stirrup harness 的可编辑面（见第 2 节）；每个候选 = 一个范围受限的 git diff。 | §3.1；§3.4 |
| 对比条件 | 默认 Stirrup（baseline）vs StarHarness (Stirrup)；描述性参照：GEPA (Pi)、独立 Pi、独立 Codex；模型维度：GPT-5.4 / GPT-5.4-mini / GPT-5.5 / Qwen3.5-27B / Qwen3.6-27B 各自 baseline vs 冻结演化 harness。 | §4.1；§4.2；Table 1 |
| 消融条件 | 无组件级消融；仅在 EnterpriseOps-Gym 上做树搜索→爬山的"探索–利用"顺序控制实验（作者自认非因果对照）。 | §3.3；§5 |
| 结果指标 | 全基准分（ITBench task score；EnterpriseOps task success；AutomationBench 目标达成份额，guardrail 违规记 0）；evolution set 与 held-out 的绝对增益；每任务估计 API 费用。 | §4.1；Table 2；Tables 3–5 |
| 控制条件 | 演化时模型权重与基准固定；迁移模型不重演化、使用同一 harness 工件；proposer 看不到 selection / holdout；按 Artificial Analysis 的评估描述计分。 | §3.1；§3.2；§4.1；§4.3 |
| 过程统计量 | 接受补丁数（4 / 12 / 5）；turns per task、shell calls、tool calls、false/true positives、verifier pass rate、mean partial credit、guardrail 违规任务数与总数、零分任务数。 | §5；Tables 3–5 |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索循环：跑种子 harness 记录分数与失败层 → 构造 search / select / holdout → 每轮：加载 ledger、收集 search 轨迹 → proposer 输出补丁 → scope / leakage / import / smoke 检查 → proposer 自选单任务 test flip → 隐藏 selection 集评估 → 按接受规则提交或回滚、更新 ledger、刷新轨迹 → 结束后在 holdout 与全基准各评估一次。 | §3.1；Alg. 1 |
| 关键设计：按 baseline 失败模式 / 分数 / verifier 通过率分层抽样紧凑评测池；search 与 selection 任务分离以区分搜索表现和泛化；持久 memory ledger 跨迭代携带 frontier 分数、逐题结果、已接受假设与被弃尝试；guardrails 禁止 task-ID 分支、验证器内容进 prompt、ground truth 访问；每个候选是可回滚的 git diff。 | §3.1；§3.2；§3.4 |
| 接受/选择规则：候选必须严格提高 selection 均分，或持平且可用的验证器指标提升，才提交为新 frontier；否则恢复原 frontier。树搜索中各节点在同一隐藏 selection 集打分，最优存活节点成为后续 frontier。 | §3.1；§3.3；Alg. 1 第 11 行 |
| 规模：三个基准各一次演化；共接受 21 个补丁（ITBench 4、EnterpriseOps 12 = 树搜索 8 + 爬山 4、AutomationBench 5）；预算 B、候选总数：未说明。 | §5 |
| 评估设计：全基准单次评估（40 / 103 / 100 题）；Table 2 对比 evolution set 与 held-out 的增益；Table 1 冻结迁移到 4 类模型；Tables 3–5 配对轨迹指标与费用。 | §4.2–§4.4；§5.1 |
| 统计显著性检验 | 未说明 |
| 重复次数 / 随机种子数 | 未说明 |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 1. 在三个基准上，harness 演化把 GPT-5.4 的全基准分提高 20–35 pp：ITBench 40.0%→75.0%、EnterpriseOps-Gym 23.3%→43.7%、AutomationBench 57.1%→83.2%，每环境接受 4–12 个改动。 | Abstract；Table 1 |
| 2. StarHarness (Stirrup) 在三个基准上都是最强配置，相对 GEPA (Pi) 高 +13.8 / +22.3 / +17.6 pp；作者称此比较为描述性。 | §4.2 |
| 3. 增益同时伴随每任务推理费用下降：ITBench −17%、EnterpriseOps −53%、AutomationBench −29%。 | §4.2；Tables 3–5 |
| 4. 冻结 harness（不重演化）对表内所有迁移模型都有正增益：GPT-5.5 +27.9 / +10.7 / +25.3 pp；GPT-5.4-mini +46.3 / +17.5 / +40.4 pp；Qwen +44.4 / +20.6 / +27.3 pp。 | Table 1；§4.3 |
| 5. held-out 任务上增益保持：+31.7 / +15.1 / +29.3 pp（evolution set 为 +45.0 / +22.0 / +23.0）。 | Table 2 |
| 6. 接受的补丁归为三类：interface repair（MCP 参数处理、schema 收窄与丰富、剥离 null/空/占位参数、结构化行操作）、environment conventions（执行契约、耦合字段更新、日期锚定、变更前 triage）、operational knowledge / search compression（forensics overview、日期与财务计算器）。 | §5 |
| 7. Codex 在 EnterpriseOps 得 41.7%（StarHarness 43.7%），其默认 MCP 预处理已能规范化 schema、减少对严格 Docker 后端的无效调用；StarHarness 学到的是互补修复。 | §5 |
| 8. ITBench：turns 25.2→22.1，shell calls 49.8→46.7（相当），false positives 0.79→0.33，true positives 0.45→0.78；回归发生在找到近因后上游搜索仍继续的情形。 | Table 3；§5.1 |
| 9. EnterpriseOps：turns 18.12→9.87，tool calls 29.53→16.83，verifier pass 34.5%→72.8%。 | Table 4 |
| 10. AutomationBench：mean partial credit 67.3%→86.1%，turns 16.35→11.98，guardrail 违规任务 20→4（总违规 33→4），零分任务 24→6。 | Table 5 |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 前提：目标是"有状态的企业环境"中工具密集型任务，存在可编程的最终状态验证器（SQL / 程序化断言 / 结构化诊断评分）给出任务分。 | §1；§2.2；§4.1 |
| 前提：harness 是外部可编辑代码（Stirrup），模型权重不可改也不需改；编辑可作为普通代码变更测试与回滚。 | §2.1；§3.1 |
| 需要先做一次全量 baseline 运行以得到失败模式、分数、verifier 通过率，再分层抽样 K≈N/2 的评测池并拆分 search / selection / holdout。 | §3.2 |
| 需要 proposer 模型（GPT-5.4）和一个能执行补丁、校验、评估的 coding harness（Oh My Pi）。 | §3.1；§4.1 |
| 所需算力/费用：演化本身的预算、token 与费用未说明；每个候选需单任务 smoke test、test flip、selection 集评估；评估期每任务费用 $0.10–$3.26（GPT-5.4，视基准）。 | Alg. 1；Tables 3–5 |
| 作者建议：harness 演化是模型扩展的实用补充；一次演化的 harness 可冻结迁移到其他模型而无需重演化；未来可用 RL 联合演化 harness 与权重。 | §6 |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| （来自作者）与 GEPA (Pi) / Pi / Codex 的比较是描述性的：系统在 prompt、工具、执行策略、架构上都不同，无法分离单一因果组件。 | §4.2 |
| （作者）无法从配对轨迹记录分离单个补丁或单个工具的因果贡献。 | §5.1 |
| （作者）树搜索→爬山为顺序阶段，描述两种模式如何互补，不是因果的 head-to-head 比较。 | §3.3 |
| （作者）AutomationBench 用的 Finance-100 子集与 Stirrup harness 与原基准论文默认设置不同，分数不可直接比较。 | §4.1 |
| （作者）Claude 参考分（Fable 5 48.1%、Sonnet 5 35.9%、Opus 4.8 max 35.5%）是外部 Artificial Analysis 参考，不属于受控的 baseline–StarHarness 比较。 | §4.3；Fig. 3 |

| （核对后发现）Table 1 中 GPT-5.5 在 EnterpriseOps-Gym 使用 high reasoning，而其他 GPT 行均为 medium；该行的 +10.7 pp 与其他行不是同一 reasoning 设置下的比较，原文未解释。 | Table 1 |
| （核对）Table 1 中 ITBench 的 Qwen 模型是 Qwen3.5-27B，另两个基准是 Qwen3.6-27B | Table 1 |


