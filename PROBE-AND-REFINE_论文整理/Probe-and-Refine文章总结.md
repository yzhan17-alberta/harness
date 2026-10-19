# Probe-and-Refine 文章总结

> 论文：*Probe-and-Refine Tuning of Repository Guidance for Coding Agents*（Asa Shepard, Jeannie Albrecht；Williams College；arXiv:2606.20512，2026 年 6 月）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：仓库级 guidance 文件（AGENTS.md 类）是否帮助 coding agent？作者主张决定变量是 guidance 如何产生——单次生成 vs 以合成失败驱动的迭代精炼。 | Abstract, §1 |
| 子问题 1：增益来自 coverage（产出可评测 patch 的实例比例）还是 precision（可评测 patch 中解决的比例）。 | §4 |
| 子问题 2：精炼 guidance 到底帮了哪类实例（定位假设）。 | §5 |
| 子问题 3：步数预算如何调节 guidance 的效果。 | §6 |
| 子问题 4：方法是否跨模型泛化；guidance 编码的是可迁移的仓库知识还是模型特定校准。 | §7 |
| 动机性问题：窄信号（少量合成 probe）能否在 prompt 层诱发对大量无关任务的广泛行为改变（类比 narrow fine-tuning）。作者声明不作机制等价主张。 | §1, §2, §8 |
| 形式化目标：无公式。隐含的分解为 resolve = coverage × precision。 | §4.4 |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被优化/被测量对象：每仓库一个 ≤3000 字符的 guidance 文本 artifact。 | §1, §3.2 |
| 方法本身：probe-and-refine tuning——每轮 4 类单次 LLM 调用（生成 10 个 probe → 每 probe 单次尝试 → 每 probe 单次判官 → 单次聚合诊断），编辑机械应用；无 agent loop、无工具、无 RL、无梯度。 | §3.3, Fig 1 |
| 任务领域：Python 仓库 bug 修复（SWE-bench Verified）。 | §2, §4.2 |
| 底层模型：Qwen/Qwen3.5-35B-A3B（MoE，3B 激活）；跨模型实验用 NVIDIA-Nemotron-3-Nano-30B-A3B（MoE，约 3.5B 激活）。 | §3.1, §7.2 |
| 优化器/判官/专家模型：与消费模型相同——probe 生成、尝试、判官、聚合均由 Qwen3.5-35B-A3B（Nemotron 实验中由 Nemotron）完成；无更强的外部专家。 | §3.3, §7.3 |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 评测 | SWE-bench Verified 500 实例，12 仓库：django 231、sympy 75、sphinx 44、matplotlib 34、scikit-learn 32、astropy 22、xarray 22、pytest 19、pylint 10、requests 8、seaborn 2、flask 1；Django 占 46%。 | Table 3, §4.2 |
| 调参 | 每仓库每轮 10 个自生成 probe（温度 0.9，跨轮去重，不足则补生成）；Qwen 共 341 个 probe、175 条编辑，Nemotron 165 / 78。不使用任何 SWE-bench 实例。 | §3.3, Table 10, Fig 1 |
| 起点 | static_kb：tree-sitter 结构摘要 + 通用建议，平均 1,687 字符（1,158–2,060）；精炼后平均 2,754 字符（1,935–2,972），长 63%。 | §3.2, Table 1 |
| 去污染 | guidance 从单一 pinned commit 生成；probe 与评测实例分离；模型预训练污染无法排除，但三条件同模型。 | §3.1, §3.3, §4.2 |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | guidance 文本的 section 内容（每轮 ≤5 条插入/修改/强化编辑，受 3000 字符上限裁剪）。 | §3.3 |
| 对比条件 | guidance 条件：no_context / static_kb / probe_refined；步数预算：25 / 50 / 100 / 200；模型：Qwen vs Nemotron；guidance 来源：Nemotron 自调 vs Qwen 调出后迁移。 | §3.2, §6.2, §7.2 |
| 消融条件 | static_kb 作为"去掉迭代精炼"的消融（结构层 + 通用建议）；未做长度 padding/truncation 消融；未做 file-structure 剥离消融。 | §3.2, §9, §6.4 |
| 结果指标 | resolve rate；coverage（可评测 patch 的实例比例）；precision（resolved/evaluated）；fallback 率；独占一致解（≥3/4 trials）；100 步后产出的 agent-loop patch 占比；prompt tokens。 | §4.3, §4.4, Table 4, Table 6, Fig 5 |
| 控制条件 | 同一 scaffold、同一评测 harness、同一硬件；温度 0.0（probe 生成 0.9 除外）；16k 有效上下文、2048 输出 tokens、fallback 512 tokens、命令输出截断约 3000 字符；guidance 一次生成后在所有条件下固定。 | §3.1, §3.3, §4.2 |
| 过程统计量 | 轮数、probe 数、编辑数、字符数（Table 1、10）；每实例平均步数、撞步数上限实例数、patch-apply 失败数、no-command prose 事件数、repeated-command stall 实例数、fallback patch 平均长度（901 vs 280 字符）。 | Table 10, Table 12, §7.4 |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索/分析循环：static_kb → 每轮 [生成 10 probe → 每 probe 单次尝试 → 每 probe 单次判官（strong/partial/missing）并提 per-probe 编辑 → 单次聚合编辑] → 合并去重、≤5 条、机械应用 → 下一轮；最多 5 轮，连续两轮无变化或无 probe 存活则早停。 | §3.3, Fig 1 |
| 关键设计：调参模型 = 消费模型，避免强弱模型能力错配；全程单次调用、无 agent loop，以便把增益归因于 instruction 质量；3000 字符上限（约占上下文 5%）；static_kb 两层设计同时控制结构信息与通用建议两个混淆。 | §3.2, §3.3 |
| 接受/选择规则：编辑不经评测验证，直接机械应用（插入/修改/强化、过滤 boilerplate、裁剪最长 bullet）；不存在基于分数的接受门控。 | §3.3 |
| 规模：12 仓库，每仓库 3–5 轮 × 约 22 次调用，每次约 8k 输入 / 约 2k 输出 tokens；新增共 104 行。 | §3.3, Table 1, Table 2 |
| 评估设计：3 条件 × 500 实例 × 200 步 × 4 次独立 trial；25/50/100 步各单 trial；Nemotron 自调三条件 + 跨模型迁移各单 trial。Agent 为 ReAct-style bash 循环，超预算后单次 fallback；patch 清洗后交官方 harness。 | §3.4, §4.2, §6.2, §7.2 |
| 统计显著性检验 | 混合效应 logistic 回归 resolved ~ condition + (1\|instance) + (1\|trial)，n=6000；每 trial McNemar；Wilson 95% CI；precision 在可评测子集（n=2971）上做似然比检验 χ²(2)=4.26，p=0.119；低预算与 Nemotron 比较用单 trial McNemar。 | §4.2, Table 5, §4.4, §6.3 |
| 重复次数 / 随机种子数 | 主实验 4 次独立 trial（200 步）；其余全部单 trial；guidance 每仓库只生成一次，probe 随机种子敏感性未测。 | §4.2, §6.2, §7.2, §9 |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 1. 4 trials、200 步：probe_refined 33.0±1.8% > static_kb 28.3±1.4% > no_context 25.5±2.2%，ordering 在 4/4 trials 成立；两条 probe 对比 p<0.001（OR 2.11 vs none、1.58 vs static），static vs none OR 1.34（p=0.004）。 | Table 4, Table 5, §4.3 |
| 2. 7.5 pp 总增益中 2.8 pp 来自 static_kb（约 37%），4.7 pp 来自迭代精炼。 | §4.3 |
| 3. 增益全部来自 coverage：56.2% vs 41.7%（+14.5 pp，约 72 个实例/trial）；precision 约 59%、条件间无显著差异（p=0.119）；按恒定精度预测约 42 个新增解，观测约 38 个。 | §4.4, Fig 4, Table 8 |
| 4. fallback 几乎无用：agent-loop patch 精度 60.1%（1,729/2,875），fallback patch 5.2%（5/96）；probe_refined fallback 率最低（14.8% vs 25.6% none、30.8% static）。 | §3.4, §4.4 |
| 5. 精炼 guidance 使 agent 用好后期步数：100 步后产出的 agent-loop patch 占比 none 4.0%、static 9.6%、probe 12.6%。 | §4.4, Fig 5 |
| 6. 独占一致解：probe 31、none 8、static 4；easy core 65、hard core 342；31 个独占解比基准平均更"易"（中位 +5 行 vs 4，45% 为 <15 分钟修复 vs 39%），共性是 API 名与实际修复文件错位；布局规整的 django（9 观测 / 14.3 期望，0.63）、pytest（0/1.2）受益少，scikit-learn 2.02×、xarray 2.20× 受益多。 | Table 6, §5.3, §5.4, Fig 6 |
| 7. 步数预算调节：25 步时三条件相当（21.8–24.4%，p>0.3）；static_kb 在 50 步激活（p=0.004）后平台；probe_refined 50 步时 23.4% 反低于 static 29.8%（p=0.022，"workflow-budget mismatch"），100 步 30.8%，200 步 33.0%；no_context 在 22.8–27.6% 平坦（25 vs 200 p=0.551）且多耗 2.3× tokens。 | Table 9, §6.3 |
| 8. Nemotron 跑不通循环：3.8/5 轮、165 probe、78 编辑（Qwen 4.5 / 341 / 175），5/12 仓库第一轮后零编辑；自调结果 no_context 28.4% > probe_refined 27.0% > static_kb 24.6%，所有 guidance 低于无 guidance。 | Table 10, Table 11, §7.3 |
| 9. 跨模型迁移崩溃：Qwen guidance 给 Nemotron 得 13.2%（66）vs 自调 27.0%（135）；机制为 "compliance by analysis"——no-command prose 事件 2,141 vs 1,252（+71%）、stall 实例 172 vs 41、撞上限 83 vs 11、fallback 324 vs 3、324 个 fallback patch 无一解决；agent-loop precision 37.9% vs 38.5% 不变。作者据此称 guidance 编码的是模型特定行为校准而非可迁移仓库知识。 | Table 12, §7.4 |
| 10. 对实践者：为将处理大量 issue 的仓库投资迭代精炼的 guidance；步数预算要容纳所规定的工作流；用将要消费 guidance 的模型来调参。 | §8, §10 |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 仓库会被 agent 反复处理（artifact 一次生成、跨 issue 复用才划算）。 | §3.3 |
| 调参模型必须能产出足够丰富的诊断性 patch 尝试并做出可操作的评判，否则循环停滞（Nemotron 案例）；循环 prompt 目前按 Qwen 输出格式设计。 | §7.3, §9 |
| 三方匹配：guidance 的工作流深度、步数预算、模型能力需相称；步数不足（如 50 步）时精炼 guidance 反而低于简单 guidance；本文 100 步以上才显现优势。 | §6.3, §8 |
| 用将要消费 guidance 的模型调参，不要跨模型复用。 | §7.4, §8, §10 |
| 单文件 ≤3000 字符的形式不解决超大仓库（约 >100k 行）的扩展问题。 | §2 |
| 所需算力/费用：每仓库 3–5 轮 × 约 22 次单次调用，每次约 8k 输入 / 约 2k 输出 tokens（推断：按此约 0.7–1.1M tokens/仓库，原文未给总数）；推理时 prompt tokens 比无 guidance 多约 56%，解决实例多 29%；美元、GPU、wall-clock 未说明。 | §3.3, §4.3 |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| 作者写到：方差刻画——只有 200 步做 4 trials，25/50/100 步与 Nemotron 全部单 trial。 | §9, §6.2, §7.2 |
| 作者：长度混淆——精炼 guidance 长 63%，未做 padding/truncation 消融，"模型只是受益于更多 prompt tokens"的可能性仍开放。 | §9, §3.2 |
| 作者：只在一个能跑通循环的模型上验证；Nemotron 失败可能部分源于 prompt 不匹配；其他 scaffold（上下文摘要、树搜索）可能不同。 | §9, §7.3 |
| 作者：probe 集/随机种子敏感性未测；Django 占 46%，非 Django 仓库最少 1–2 实例，无法做 per-repo 显著性检验；file-structure 消融（定位假设的直接检验）未做；跨模型只测 incidental transfer。 | §9, §6.4, §7.4 |
| 覆盖范围：只测强→弱、跨家族单方向迁移，无同家族版本升级、无 weak→strong 反向；无采样数（best-of-N）对照，只扫步数；无人工撰写 guidance 对照；16k 截断使绝对值偏低（22–35%，前沿系统 >55%）。 | §7, §6.2, §8 |


