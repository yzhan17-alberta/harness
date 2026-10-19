# STARHARNESS 对照表

论文：*StarHarness: Evolving Harnesses with Stratified Search for Enterprise Environments*（Esakkivel Esakkiraja, Denis Akhiyarov, Vikas Yadav, Sai Rajeswar, Patrice Bechard, Sridhar Nemala, Sagar Davasam；ServiceNow / Mila / Université de Montréal；arXiv:2608.24804，2026 年 8 月）

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 离线、部署前：在固定模型外做"外层循环优化"，演化出环境专属 harness；演化完成后冻结，再在 holdout 和全基准上评估一次，并原样迁移到其他模型。 | §3.1；Alg. 1 第 13 行；§4.3 |
| 适用环境 | 三个企业基准：ITBench SRE（40 个 Kubernetes 根因分析场景，离线事故快照）；EnterpriseOps-Gym ITSM（103 个 ITSM oracle 任务，取自 1,150 题全集，ServiceNow MCP 后端，SQL 验证器判终态）；AutomationBench Finance（100 个财务流程任务，47 个模拟 SaaS 应用，程序化断言判分，违反 guardrail 记 0 分）。 | §2.2；§4.1 |
| 轨迹来源 | 被评测的 agent 是 GPT-5.4（medium reasoning）跑在 Stirrup harness 上；先用种子 harness 在全部 N′ 个可复现任务上跑一遍 baseline 记录分数和失败层；之后每轮从当前 frontier harness 的 search 集轨迹中取材。 | §3.1；§3.2；§4.1；Alg. 1 第 1、5 行 |
| 可见轨迹 | Proposer 可读：当前 frontier harness、持久 memory ledger、search 集的轨迹与结果；不可读：selection 集和 holdout 集的任务内容、轨迹、验证器反馈、逐题结果。 | §3.1；Fig. 1 说明；§3.2 |
| 交给优化器的输入 | 每轮 Δ ← P(h, T, L)：h = 当前 harness，T = search 轨迹，L = ledger（frontier 分数、逐题结果、已接受假设、被丢弃尝试；树搜索时还含候选节点、父指针、失败状态、晋升决定）。 | §3.1；Alg. 1 第 5–6 行 |
| 诊断依据 | Proposer 自行检查 search 任务的轨迹和评估结果来诊断重复失败；分数由各基准验证器给出（ITBench 结构化诊断评分、EnterpriseOps SQL 验证器、AutomationBench 程序化断言）；候选还要过 validator（scope、imports、单任务 smoke test）和 proposer 自选的单任务 test flip。 | §3.1；§3.4；§4.1 |
| 关注的失败 | 分层抽样用的 baseline 失败模式：wrong_tool、context_loss、missing_evidence、premature_conclusion；分析阶段归纳为三类模型–环境摩擦：interface repair、environment conventions、operational knowledge / search compression；轨迹指标还统计 false positives、guardrail violations、zero-score tasks。 | §3.2；§5；Tables 3、5 |
| 禁止使用 | 禁止：按 task ID 分支或硬编码答案；把验证器 / 断言内容写进 agent prompt；访问 ground-truth 表或隐藏状态；基准特定的答案映射。Proposer 不得看到 selection / holdout 的内容与结果；holdout 从不影响提案或接受。 | §3.4；§3.2 |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 修改被评测的 Stirrup agent harness 的全部可编辑面（prompt/任务框架、工具定义与 schema、参数预处理、skills、MCP providers、subagent 结构、上下文管理、验证、finish 逻辑）；模型权重与基准固定。每个候选是对 frontier 的 git diff，范围限定在该基准的可编辑目录与共享 agent 框架。 | §3.1；§3.4 |
| prompt / 工具描述 | 可改 prompt 与任务框架、工具定义与 schema、参数预处理。实例：EnterpriseOps 修复 MCP 参数处理、保留复合 schema、删掉误导字段、加链接与自引用提示；收窄并丰富 schema，调用前剥离 null / 空 / 占位参数；AutomationBench 用结构化行操作替代原始电子表格编辑；系统指引把相对日期锚到沙盒时钟。 | §3.1；§5 |
| skill / 记忆 | skills 属可编辑面。实例：ITBench 新增"forensics overview"按观测证据给候选上游原因排序；AutomationBench 新增日期与财务计算器。memory ledger 属优化器一侧（跨迭代记录），不是 agent 运行时记忆；原文未说明 agent 侧是否加了记忆组件。 | §3.1；§5 |
| 控制逻辑（执行前） | 可改参数预处理、MCP providers、subagent 结构。实例：变更前先做 triage 步骤；执行契约（execution contract）；耦合的 priority 与 impact/urgency 更新；要求保留关系字段。 | §3.1；§5；§5.1 |
| 控制逻辑（执行后） | 可改上下文管理、验证、finish 逻辑。实例：execution and finish contracts 针对未完成流程；自引用与链接提示在依赖变更间保持状态。 | §3.1；§5.1 |
| 明确不改 | 模型权重、基准本身、任务数据与验证器均不改。 | §3.1；§5 |
| 对照组 | 基线 = 未修改的 Stirrup 框架（默认 prompt 与工具配置）；另外描述性比较独立的 Pi harness、Codex harness，以及 GEPA prompt 优化叠加在 Pi harness 上（GEPA (Pi)）。 | §4.1；§4.2；§5 |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | 先用种子 harness 在全部 N′ 个任务上跑一次 baseline；评测池 K≈N/2，再分成 search / selection；holdout = N′−K。预算 B（迭代轮数）的具体取值、总 rollout 数、token 数：未说明。 | §3.2；Alg. 1 |
| 诊断 | Proposer 为 GPT-5.4，运行在基于 Oh My Pi 的 coding harness 中；调用次数、token、费用：未说明。 | §3.1；§4.1 |
| 搜索补丁 | 三次演化共接受 21 个补丁：ITBench 4、EnterpriseOps-Gym 12（树搜索阶段 8 + 爬山阶段 4）、AutomationBench 5。被拒 / 无效 / 崩溃候选数、演化总费用与时长：未说明。 | §5 |
| 验证 | 每个候选依次过 validator（scope、imports、单任务 smoke test）→ proposer 自选的单任务 test flip（不翻转则跳过昂贵评估）→ 隐藏 selection 集评估；结束后在 holdout 与全基准各评估一次。评估期每任务推理费用（GPT-5.4）：ITBench $3.26→$2.70（−17%）、EnterpriseOps $1.23→$0.58（−53%）、AutomationBench $0.14→$0.10（−29%）。 | §3.1；Alg. 1；Tables 3–5；§4.2 |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | GPT-5.4 全基准分：ITBench 40.0%→75.0%（+35.0 pp）；EnterpriseOps-Gym 23.3%→43.7%（+20.4 pp）；AutomationBench 57.1%→83.2%（+26.1 pp）。相对 GEPA (Pi)：+13.8 / +22.3 / +17.6 pp（Fig. 2，绝对值未在正文给出）。轨迹层面：ITBench false positives 0.79→0.33、true positives 0.45→0.78；EnterpriseOps turns/task 18.12→9.87、tool calls 29.53→16.83、verifier pass 34.5%→72.8%；AutomationBench guardrail 违规任务 20→4、零分任务 24→6、mean partial credit 67.3%→86.1%。 | Table 1；Tables 3–5；§4.2 |
| 对其他任务是否有效 | 同基准内 held-out（演化中从未使用）增益：ITBench +31.7（evolution set +45.0）；EnterpriseOps +15.1（+22.0）；AutomationBench +29.3（+23.0）。跨基准迁移：未说明（每个基准各自演化）。 | Table 2；§5 |
| 回归：流程层面 | 有确定性门控：候选必须严格提高 selection 均分，或持平且可用的验证器指标提升才提交；否则回滚到前一 frontier。前置还有 validator 与 test flip 两道门。 | §3.1；§3.3；Alg. 1 第 11–12 行 |
| 回归：结果层面 | 未报告逐题回归计数、重复次数、方差或显著性检验；所有表为单次数值。仅定性指出 ITBench 的回归发生在"找到近因后上游搜索仍继续"的情形。 | §5.1；Tables 1–5 |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 用 GPT-5.4 演化的 harness 冻结（不重演化）迁移。同族升级 GPT-5.5：ITBench (medium) 50.8→78.7（+27.9）；EnterpriseOps (high) 37.8→48.5（+10.7）；AutomationBench (medium) 59.6→84.9（+25.3）。同族降级 GPT-5.4-mini (medium)：33.1→79.4（+46.3）；13.6→31.1（+17.5）；29.6→70.0（+40.4）。跨家族 Qwen：ITBench Qwen3.5-27B 25.6→70.0（+44.4）；EnterpriseOps Qwen3.6-27B 18.2→38.8（+20.6）；AutomationBench Qwen3.6-27B 48.2→75.5（+27.3）。表中只有 GPT 行标注 reasoning 等级。 | Table 1；§4.3 |
| 换任务 | 同基准内 held-out 任务上增益保持（见第 4 节）；跨基准 / 跨领域：未说明。 | Table 2 |
| 换执行环境 | 未说明（三个环境各有各自演化的 harness）。 | §5 |
| 换 Harness 实现 | 未说明；全部演化与迁移实验都在 Stirrup 上。Codex（独立 harness）在 EnterpriseOps 得 41.7%，仅作参照，不是把演化补丁移植到 Codex。 | §4.1；§5 |
| 补充 | 作者写到：与 GEPA (Pi) / Pi / Codex 的比较是描述性的，无法分离单一因果组件；无法从配对记录分离单个补丁或单个工具的因果贡献；树搜索→爬山是顺序阶段，不构成因果对照；AutomationBench 子集与 harness 与原基准论文不同，分数不可直接比较；未来方向是用 RL 联合演化 harness 与权重。 | §4.2；§5.1；§3.3；§4.1；§6 |
