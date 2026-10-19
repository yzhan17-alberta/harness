# COEVOLVE 对照表

论文：*Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails*（Zhou Yu, Bin Bi, Shiva Kumar Pentyala, Shubham Mehrotra, Sougata Chaudhuri, Shilpa Bhagavath, Zeyuan Chen, Ran Xu, Phil Mui, James Zhu, Sitaram Asur；Salesforce AI；arXiv:2609.09134，2026 年 9 月）

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 离线、部署前，分两步：先用弱模型演化 harness（GEPA-style 搜索），再在冻结的演化后 harness h* 下做 LoRA-SFT 改模型权重；作者将其描述为可多轮迭代的 co-evolution loop，但实验只做了一轮。训练/验证集用于优化，测试集 held-out。 | §3.1, §3.4, Fig 1, App. A |
| 适用环境 | 7 个可客观验证的企业 agentic 任务（沿用 Yang et al. 2026 的任务套件与环境）：payroll auditing（表中 Attn/Attendance）、budget approval、stock alerting、IoT anomaly detection、browser automation（Playwr）、website management（Web，正文又称 Webarena）、code refactoring。每任务测试集大小未说明。对环境做了两处改动：需要 MCP 工具时 agent 不能直接改任务数据库；tool-call 错误暴露给优化器。 | §3.1, Table 1 |
| 轨迹来源 | harness 演化：弱模型 qwen3-coder-30b-a3b（Gemma 复现用 gemma-4-26b-a4b-it）作执行器，在 train/val 上 rollout。SFT-imit 数据：gemini-3.1-pro-preview 专家在 h* 下得分 1.0 的轨迹 + 弱模型自己在 h* 下通过的轨迹。SFT-corr 数据：弱模型自己在 h* 下的 rollouts（失败的由专家改写单个 turn，另混入 ~50 行自通过轨迹）。 | §3.1, App. A, App. B |
| 可见轨迹 | harness 优化器：每轮反思一个 minibatch 上的失败（failure-driven edits），tool-call 错误可见。SFT-corr：自动 failure-locus 步骤看弱模型的完整 rollout 并定位出错的单个 turn；专家看任务说明、到该 turn 为止的轨迹、以及失败的 checkpoint。失败分析：LLM-as-judge 看每条失败 rollout 的失败 checkpoint 与工具错误。 | §3.1, App. A, App. B, App. C |
| 交给优化器的输入 | harness proposer（gemini-3.1-pro-preview meta-agent）：minibatch 上的反思，输出一条 harness 编辑。专家改写器：任务说明 + 该 turn 之前的轨迹 + 失败的 checkpoint，要求只重写该 turn（一句修正策略 + 修正后的 tool call），采样 N=3 个候选，由 quality judge 选最佳。 | §3.1, App. A, App. B |
| 诊断依据 | harness 演化：编辑只有在该 minibatch 上提升性能才保留；最终 h* 按 validation 分数选。失败 turn 定位：自动化的 "failure-locus" 步骤（MLE agent）。失败归类：LLM-as-judge 分类器。任务成功为二值、可客观验证。 | §3.1, App. A, App. B, App. C |
| 关注的失败 | 沿用 Yang et al. 2026 的六类 adaptation-failure ontology：0 other、1 tool-use、2 instruction-following、3 implicit knowledge、4 long-context、5 planning。分析聚焦 implicit knowledge（答错领域值、漏掉隐含约定、跳过从业者会做的步骤）与 planning（漏关键子任务、错误计划不回退、无进展的重规划循环）。 | §3.3.3, App. C, Table 3 |
| 禁止使用 | 测试集 held-out，只有训练/验证集用于优化。环境层面：需要 MCP 工具时 agent 不能直接修改任务数据库。其余（如 task id、ground truth 是否对 proposer 可见）未说明。 | §3.1 |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 两个杠杆都改：harness（system prompt、tool set、execution hooks、context management、sub-agent 配置）通过自动演化修改；模型权重通过 LoRA-SFT 修改。研究问题是两者如何组合。 | §1, §3.1, Abstract |
| prompt / 工具描述 | 可编辑组件主要是 agent 的 system prompt，proposer 也可增删 tools 与 hooks。Table 4 列出各任务胜出编辑：Attendance（3a, 1c）写明聚合与薪资规则、加打不开文件的 workaround；Anomaly（3a, 1c）一次重写给出 query-detect-report-upload 全流程；Stock（1a, 3a, 2c）新增枚举全目录的工具并要求处理每个低库存项；Playwright（3a）写明 sandbox 约定并要求完成前本地验证；Website（1a, 3a）新增 structured finish tool 并外化 admin-console 约定；Budget（3a, 5a/5b）外化核心协议、加显式有序计划与 state-access guard；Refactor 插入全局搜索工具。 | App. A, Table 4 |
| skill / 记忆 | 未说明（§3.1 列出 context management 与 sub-agent 配置为可优化项，但未给出任何此类编辑的实例）。 | §3.1 |
| 控制逻辑（执行前） | hook 级强制：Table 2 的 "Enforce via hook" 类编辑（用 hook 阻断被禁止的动作，ontology 2d）；Budget 任务的 state-access guard。 | Table 2, Table 4 |
| 控制逻辑（执行后） | 优化时每个 rollout 600 s 时限。Playwright 编辑要求"完成前本地验证"；Case 1 的 h* 含单次输出校验步骤（打印最终列并核对 schema 后再保存）。其他监控/循环检测/预算机制未说明。 | App. A, Table 4, App. D |
| 明确不改 | harness 演化阶段不改权重；SFT 阶段 harness 冻结（在同一 h* 下训练与评估）。SFT 只用 LoRA（rank 16 或 64），不做全参训练或 RL（RL 列为未来工作）。 | §3.4, App. B, §4 |
| 对照组 | Table 1 七行：弱模型 base 在 h0 / h*；专家 gemini-3.1-pro-preview 在 h0 / h*；SFT-imit 在 h* / h0；SFT-corr 在 h*。Gemma 复现只有 base h0 / h* / SFT-imit h* 三个数（Webarena 单任务）。 | Table 1, §3.3.2 |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | harness 演化：每任务用三个随机种子、$20 预算、600 s/rollout 时限（原文为 "three random seeds with a budget of $20"，$20 是每个种子还是三个种子合计未说明）。SFT-corr 训练集约 500 行（约 400 行专家原地修正 + 约 50 行自通过）。SFT-imit 训练集大小未说明；验证集 rollout 数未说明。 | App. A, App. B |
| 诊断 | 未说明（failure-locus 定位与 LLM-as-judge 分类的调用次数/token 未报告）。 | — |
| 搜索补丁 | harness 搜索预算见上；专家改写每个失败 turn 采样 N=3 候选 + quality judge，token 成本未说明。 | App. A, App. B |
| 验证 | 训练：单次 LoRA run 不到 1 小时，H100/H200 GPU。推理：合并后的稠密 checkpoint 约 57–61 GB，在 2 块 H200 上 tensor parallel。测试结果为 3 次运行平均。评测总 rollout 数与费用未说明。 | §3.1, §4, App. B |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | 指标：测试集 task success（%）。h0→h*：29.2→78.0（+48.8）。SFT-imit 在 h*：78.0→63.1（−14.9），7/7 任务回退，范围 −4.2（Anom）到 −29.9（Attn）。SFT-corr 在 h*：78.0→79.7（+1.7），5/7 任务提升（Web +5.6、Stock +2.2、Refact +2.2、Anom +1.7、Playwr +1.2）。失败构成：planning 占比 h* base 1.1% → imit 14.6% → corr 1.8%；knowledge 46.2 → 44.5 → 43.2；总失败率 28.9% → 26.8%（corr）。 | Table 1, Table 3, §3.2–3.4 |
| 对其他任务是否有效 | 测试集 held-out，但每任务各自演化、各自训练，无跨任务 held-out。第二模型家族复现：Gemma 在 Webarena 上 h0 46.7 → h* 55.6 → SFT-imit 41.1。 | §3.1, §3.3.2 |
| 回归：流程层面 | harness 演化中有门控：编辑只在 minibatch 上提升才保留，保留者进入 Pareto pool，最终按 validation 分数选 h*（平局取最早迭代）。SFT 阶段无非回归门控（未说明）。 | App. A |
| 回归：结果层面 | 每格报告 3 runs 的 mean ± SEM 与相对基线的逐任务 Δ。SFT-corr 在 Budget −0.6、Attn −0.4，作者称 "within noise"；SFT-imit 在 h0 下 Playwr −1.6、Refact −2.2。无显著性检验（未说明）。 | Table 1, §3.4 |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 弱→强、harness 冻结（向上迁移）：gemini 在 h0 84.4 → h* 93.6（平均 +9.2，Table 1 行 3→4；行 4 括号内的 +15.6 是行 4 相对行 2，即专家 vs 弱模型同在 h* 的差）；专家触发几乎每条演化编辑（93.6–100% rollouts），domain-computation recipe 94.2% vs base 30.8%。Gemma 的 h* 也向上迁移到 gemini：71.1 → 81.1（+10.0）。同一模型权重经 LoRA-SFT 改变后（harness 冻结）：SFT-imit 78.0 → 63.1；Gemma 55.6 → 41.1（−14.5，比 h0 的 46.7 还低 5.6）；SFT-corr 78.0 → 79.7。未做同家族版本升级，未做 SFT 后再演化 harness 的实验。 | §3.2, §3.3.2, §3.4, Table 1, Table 2 |
| 换任务 | 每任务单独演化 harness、单独训练；跨任务迁移未说明。 | §3.1, App. A |
| 换执行环境 | 未说明。作者只指出因两处环境改动，绝对结果可能不与 Yang et al. 2026 直接可比。 | §3.1 |
| 换 Harness 实现 | 未说明（全部实验用 Yang et al. 2026 的代码库与 base harness h0）。 | §3.1, App. A |
| 补充 | 作者自认：SFT-corr 未完全关闭弱模型与专家的差距，"likely due to a limit of the lightweight LoRA approach"；未来计划结合 RL 与 on-policy 专家指导，并让 harness 演化"aware"后续会 fine-tune。未测项：co-evolution loop 的多轮迭代（Fig 1 只描述）；Gemma 只在 Webarena 一个任务且无 SFT-corr 结果；LLM judge 的一致性未报告。 | §3.4, §4, §3.3.2, App. C |
