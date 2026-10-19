# CoEvolve 文章总结

> 论文：*Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails*（Zhou Yu, Bin Bi, Shiva Kumar Pentyala, Shubham Mehrotra, Sougata Chaudhuri, Shilpa Bhagavath, Zeyuan Chen, Ran Xu, Phil Mui, James Zhu, Sitaram Asur；Salesforce AI；arXiv:2609.09134，2026 年 9 月）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：在合理预算下做任务特定的领域适配时，harness 演化与轻量模型微调（LoRA-SFT）应如何组合；两者能否 co-evolve 产生协同增益而非相互干扰。 | §1 |
| 子问题 1：为弱模型演化的 harness 是否能被更强的专家模型使用（向上迁移），从而暴露模型侧的剩余空间。 | §1 (i), §3.2 |
| 子问题 2：在演化后的 harness 下用专家轨迹做全轨迹模仿为何反而让弱模型回退，机制是什么。 | §1 (ii), §3.3 |
| 子问题 3：on-policy 专家修正（只改写弱模型自身 rollout 中出错的单个 turn）能否在不破坏 model–harness fit 的前提下带来模型侧增益。 | §1 (iii), §3.4 |
| 形式化目标：未给出公式。Fig 1 描述循环：每轮围绕当前模型 M_t 演化 harness H_t*，再在该 harness 下更新模型得到 M_{t+1}，再进入下一轮。 | Fig 1 |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被优化/被测量对象：harness（system prompt、tool set、execution hooks、context management、sub-agent 配置）与弱模型的 LoRA 权重；测量对象为 model–harness fit（以任务成功率与失败构成衡量）。 | §1, §3.1 |
| 方法本身：(a) GEPA-style harness 演化（沿用 Yang et al. 2026 框架）；(b) SFT-imit：在 h* 下用专家成功轨迹 + 弱模型自身成功轨迹做 LoRA-SFT；(c) SFT-corr：由 MLE agent 定位弱模型失败 rollout 的出错 turn、专家只重写该 turn，再做 LoRA-SFT。 | §3.1, §3.3, §3.4, App. A, App. B |
| 任务领域：7 个企业 agentic 任务（payroll auditing、budget approval、stock alerting、IoT anomaly detection、browser automation、website management、code refactoring）。 | §3.1 |
| 底层模型：弱模型 qwen3-coder-30b-a3b（训练用 Qwen/Qwen3-Coder-30B-A3B-Instruct）；第二家族 gemma-4-26b-a4b-it（仅 Webarena 任务）。 | §3.1, §3.3.2, App. B |
| 优化器/判官/专家模型：gemini-3.1-pro-preview 同时担任 harness 演化的 meta-agent/反思模型和 SFT 的专家。failure-locus 定位的 MLE agent、best-of-3 的 quality judge、失败分类的 LLM-as-judge 所用模型均未说明。 | §3.1, App. A, App. B, App. C |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 企业 agentic 任务 | Yang et al. 2026 的 7 任务套件，有 train / validation / test 切分；训练与验证用于优化，测试 held-out。每任务实例数未说明；去污染未说明。 | §3.1 |
| SFT-imit 训练数据 | h* 下得分 1.0 的轨迹：先取 gemini-3.1-pro-preview 专家的，再取 base 模型自己通过的；转换为学生模型的 chat/tool-call 格式。Gemma 复现另将 gemini 轨迹整形为 Gemma 的推理格式。规模未说明。 | §3.1, App. B |
| SFT-corr 训练数据 | 约 500 行：约 400 行原地专家修正（来自 base 模型在 h* 下的失败 rollout）+ 约 50 行自通过 rollout（优先取"有时通过有时失败"的能力边界样本）。 | App. B |
| 失败分析数据 | 每个条件在相应 harness 下所有得分为 0 的 rollout（hard failure）。 | App. C |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | harness 编辑（system prompt 为主，可增删 tools 与 hooks）；LoRA 权重（rank 16 或 64）。 | App. A, App. B |
| 对比条件 | harness：h0（baseline）vs h*（evolved）。模型：base vs SFT-imit vs SFT-corr；弱模型 vs 专家 gemini-3.1-pro-preview。 | Table 1 |
| 消融条件 | Ablation I：同样的 SFT-imit 配方放到 h0 下（排除"模仿本身有害"）。Ablation II：换成 gemma-4-26b-a4b-it 在 Webarena 上重复（排除特定 weak–strong 配对的影响）。 | §3.3.1, §3.3.2 |
| 结果指标 | 测试集 task success（%）；hard failure 中 knowledge / planning 两类的占比；总失败率；演化编辑的采用率（rollouts 中至少使用一次的比例）。 | Table 1, Table 2, Table 3, §3.4 |
| 控制条件 | 所有模型在同一 h*（为 Qwen 演化）下比较；所有 SFT 配方用同一 base 模型与 LoRA 配置；SFT 后在同一 h* 下评估。采样温度等推理参数未说明。 | Table 2, App. B, §3.4 |
| 过程统计量 | 各编辑类别采用率（Table 2）；Case 1 的步数（base 14–20 步 vs SFT-imit 78 步，重发同一 view 29 次、重读 prompt 约 20 次）。 | Table 2, App. D |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索/分析循环：harness 演化每轮在一个 minibatch 上反思 → 提出一条编辑 → 仅当该 minibatch 性能提升时保留 → 进入 Pareto pool；h* 取 validation 分数最高者。SFT-corr 流水线：弱模型在 h* 下 rollout → MLE agent 定位失败 turn → 专家重写该 turn（N=3，quality judge 选最佳）→ 混入自通过样本 → LoRA-SFT → 在同一 h* 下评估。 | App. A, App. B, §3.4 |
| 关键设计：on-policy 修正只改一个 turn、保留弱模型自身的 planning 分布，以维持 harness 为该模型 native planning 演化出的 fit；相比之下全轨迹模仿会转移专家的 planning 风格。 | §1, §3.4, §3.3.3 |
| 接受/选择规则：harness 编辑按 minibatch 提升保留，h* 按 validation 分数、平局取最早迭代；SFT 报告 rank 16 与 64 中较好的分数（选择依据的数据切分未说明）；修正候选由 quality judge 选。 | App. A, App. B |
| 规模：7 任务；harness 演化每任务三个随机种子、$20 预算、600 s/rollout；LoRA 2 epochs、lr 1e-4、bf16、有效 batch 8、序列长度 49,152（若 >5% 训练数据被截断则再训 98,304）；SFT-corr 约 500 行。 | App. A, App. B |
| 评估设计：测试集 held-out，报告 3 次运行的 mean ± SEM；失败构成用 LLM-as-judge 按六类 ontology 分类，报告为该条件失败中的占比；同一分类器做采用率扫描。 | §3.1, Table 1, App. C |
| 统计显著性检验 | 未说明（只报 SEM，"within noise" 为作者口头判断）。 |
| 重复次数 / 随机种子数 | 测试成功率为 3 runs 平均；harness 演化 3 个随机种子；SFT-imit 主结果为 3 个 LoRA 训练种子（0, 101, 202）平均；SFT-corr 的训练种子数未说明。 | §3.1, App. A, App. B |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 1. 用弱模型演化 harness 使 Qwen 平均测试成功率从 29.2% 升到 78.0%（+48.8），所有任务都提升。 | §3.2, Table 1 行 1–2 |
| 2. 该 harness 向上迁移：gemini-3.1-pro-preview 从 h0 的 84.4% 升到 h* 的 93.6%（平均 +9.2）；专家在 93.6–100% 的 rollouts 中触发几乎每条演化编辑，domain-computation recipe 使用率 94.2% vs base 30.8%；专家在 h* 上比弱模型高 15.6。 | §3.2, Table 1 行 3–4, Table 2 |
| 3. 在 h* 下模仿专家轨迹（SFT-imit）使 Qwen 从 78.0% 降到 63.1%（−14.9），7/7 任务回退，从 −4.2（Anom）到 −29.9（Attn）。 | §3.3, Table 1 行 5 |
| 4. 同一配方在 h0 下反而有效：29.2% → 35.5%（+6.3），故问题出在模仿与 harness 演化的交互而非模仿本身。 | §3.3.1, Table 1 行 6 |
| 5. Gemma 复现（Webarena）：h0 46.7 → h* 55.6（+8.9）；h* 向上迁移到 gemini 71.1 → 81.1（+10.0）；SFT-imit 后降到 41.1（比 h* 低 14.5，比 h0 还低 5.6）。 | §3.3.2 |
| 6. 回退不是因为少用 harness 或丢失知识：SFT-imit 后 domain-computation 使用率 30.8% → 76.1%，knowledge 失败占比 46.2 → 44.5（−1.7）；回退来自 planning 失败占比 1.1% → 14.6%（+13.5）。h0 下 planning 也 0.9 → 11.5（+10.6），但 knowledge 61.0 → 54.4（−6.6）主导，净成功率上升。 | §3.3.3, Table 2, Table 3 |
| 7. SFT-corr 使两个杠杆可叠加：78.0% → 79.7%（+1.7），5/7 任务提升，Budget −0.6 / Attn −0.4 在噪声内；planning 占比仅 1.1 → 1.8（+0.7），knowledge 46.2 → 43.2（−3.0），总失败率 28.9% → 26.8%。 | §3.4, Table 1 行 7, Table 3 |
| 8. 案例：payroll audit 中 SFT-imit 模型算出正确结果但把单次校验变成无界循环——重发同一 view 29 次、重读 prompt 约 20 次、78 步内从未 finish，得 0 分；Webarena 中 SFT-imit Gemma 跳过"先过滤再计数"，读未过滤总数 351（真值 346）。 | App. D |
| 9. 配方轻量：单次训练不到 1 小时，可安全叠入 co-evolution loop；设计原则是 harness 演化后用学生模型访问过的状态做 on-policy 监督，而非整体模仿强模型轨迹。 | §1, §4 |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 前提：任务可客观验证（二值得分）且有 train/val/test 切分；已有可编辑的 harness 代码库（Yang et al. 2026）与 base harness h0。 | §3.1, App. A |
| 需要一个在 h* 上明显强于弱模型的专家（本文 gemini-3.1-pro-preview）既做 harness 反思模型又做 turn 级修正。 | §3.1, App. B |
| 弱模型需可做 LoRA-SFT 且轨迹可转成其 chat/tool-call 格式；对推理型模型（Gemma）还需整形推理格式。 | §3.1, App. B |
| 所需算力/费用：harness 演化每任务三个随机种子、$20 预算、600 s/rollout；LoRA 训练单次 <1 h（H100/H200）；推理 2×H200 tensor parallel，合并后 checkpoint 约 57–61 GB。数据收集、定位、评测的 token/费用未说明。 | App. A, App. B, §4 |
| 使用建议：harness 演化之后不要整体模仿强模型轨迹，改用 on-policy、局部化的专家修正，以保持 model–harness fit；修正由 MLE agent 自动生成，可迭代叠加。 | §1, §3.4, §4 |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| 作者写到：SFT-corr 未完全关闭弱模型与专家差距（+1.7 vs 专家领先 15.6），归因于轻量 LoRA 的上限；未来需结合 RL。 | §3.4, §4 |
| 作者：harness 演化目前不知道模型之后会被微调，两臂是顺序而非联合优化；多轮 co-evolution 未实验。 | §4, Fig 1 |
| 作者：因环境改动，绝对数字可能不与 Yang et al. 2026 直接可比。 | §3.1 |
| 覆盖范围：Gemma 只在 Webarena 一个任务复现，且无 SFT-corr 结果；只测弱→强单方向迁移；没有同家族版本升级、没有 SFT 后重新演化 harness、没有 budget-matched 的 test-time-scaling 对照（均未说明）。 | §3.3.2, §3.2 |

