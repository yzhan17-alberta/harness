# Rethinking 文章总结

> 论文：*Rethinking the Evaluation of Harness Evolution for Agents*（Yike Wang, Huaisheng Zhu, Zhengyu Hu, Yige Yuan, Zhengyu Chen, Shakti Senthil, Hannaneh Hajishirzi, Yulia Tsvetkov, Noah A. Smith, Pradeep Dasigi, Teng Xiao；Allen Institute for AI / University of Washington / Independent；arXiv:2607.12227，2026 年 7 月）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：自动 harness evolution 的收益究竟是可复用的 harness 设计改进，还是对被评测任务的重复采样 / 测试时发现？现有评测协议无法区分二者。 | Abstract；§1；§3 |
| 子问题：（1）现有工作没有在匹配反馈与推理预算的条件下与简单的任务级搜索基线（parallel sampling、sequential refinement）比较；（2）现有工作用与最终报分相同基准上的验证信号（单测）来搜索 harness，违反训练/测试分离。 | Abstract；§1 |
| 形式化目标：固定 policy π_θ、任务分布 X、预算 K，每种方法由"更新什么"和"看到什么反馈"定义；harness evolution 的目标 h* = argmax_h E_{x∼X}[R(y, g)]，y ∼ π_θ(·|x; h)。 | §3；§3.1；§3.4 |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被测量对象：四种预算分配方式——Parallel Sampling（K 条独立轨迹）、Sequential Refinement（K 轮轨迹修订）、Harness Evolution（跨任务共享的 harness 更新）、Harness Scaling（单任务实例的 harness 更新）。 | §3；Fig. 1 |
| 方法本身：这是评测协议研究。Harness Evolution 用 AHE（Lin et al., 2026）实例化，关闭其 explore agent，只保留 propose–refine 循环；Harness Scaling 是作者引入的 harness 级 test-time scaling 类比，从初始 harness h_1 出发，每轮由 meta agent M 根据 Φ(上一 harness, 上一轨迹[, 结果]) 改 harness 再重采样。 | §3.4；§3.5；App. A.1 |
| 任务领域：终端编码任务（Terminal-Bench 2.1）；交互式视觉游戏（ARC-AGI-3）；真实游戏 / 模拟器的 bot 编写（EdgeBench Interactive Games & Simulators）。 | §4.1；§5.1 |
| 底层模型：Terminal-Bench 为 Claude Opus 4.6、GPT-5.4、GPT-5.4 mini（high reasoning，单轮最大生成 128k tokens）；ARC-AGI-3 为 Claude Opus 4.8（xhigh，claude-code@2.1.220）、GPT-5.6 Terra（high，codex@0.145.0）；EdgeBench 为 Claude Opus 4.8（high，claude-code@2.1.159，上下文 1,000,000 tokens）、GPT-5.4（xhigh，codex@0.130.0）。 | §4.1；Table 6；Table 10；Table 14 |
| 优化器 / 判官 / 专家模型：摘要器 Φ（Agent Debugger）、meta agent M、self judge J 均由与 policy 相同的模型实现；无外部专家模型。有单测时判官为单测结果 R(y, g)。 | §3.1；§3.2；§3.5；Table 6 |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 终端编码 | Terminal-Bench 2.1：89 个终端任务，是 2.0 的验证修订版（修复 28 个任务的外部依赖漂移、资源预算不匹配、指令与测试不一致）。§4.4 切分为 45 训练 / 10 验证 / 34 测试。去污染：未说明（§5 提到模型可能在训练期已针对 Terminal-Bench 优化）。 | §4.1；§4.4；§5 |
| 交互式视觉游戏 | ARC-AGI-3：25 个公开 demo 环境，64×64 网格、16 色，逐关卡计分；指标 RHAE = 逐关 min(1.15, (a_human/a_agent)²) 的按关卡加权均值，人类 100% 可解。 | §5.1；App. B.1 |
| 游戏 / 模拟器 bot | EdgeBench Interactive Games & Simulators 中 5 个任务：dcss_dungeon_ai（Lua bot，5 隐藏种子，每局 900 s）、nethack_dungeon_agent（agent.py，NLE）、openrct2_theme_park_ai（ES5 JS 插件）、openttd_transport_ai（Squirrel，20 游戏年 × 5 种子图）、wesnoth_tactical_ai（Lua + WML，3 地图 × 10 种子 × 2 重复 = 60 局）；0–100 分，取所有 judge 评分中的 best-so-far。 | App. C.1 |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | Harness Evolution / Harness Scaling：harness（prompt、middleware、tools、skills、长期记忆；ARC-AGI-3 上为记忆 GUIDE.md / WORKING.md、skills、tools、hooks）。Parallel Sampling / Sequential Refinement：轨迹本身，harness 固定。 | §3；App. A.2；App. B.1 |
| 对比条件 | 四种方法 vs direct sampling；有单测 / 无单测两种反馈设定；同任务评测 vs 不相交的搜索/评测任务；ARC-AGI-3 上 [Claude Code / Codex]、[VISTA*]、[VISTA]、[full harness] 四种配置；EdgeBench 上 Sequential Refinement vs Harness Scaling。 | §4.2–§4.4；Table 4；Table 5 |
| 消融条件 | 预算 K=5 vs K=10（仅 Claude Opus 4.6）；ARC-AGI-3 上记忆演化 vs 全 harness 演化（[VISTA] vs [full harness]）。 | App. A.3；Tables 7–9；§5.2 |
| 结果指标 | pass@1（Eq. 1，基础设施异常记 0）、pass@5 / pass@10（Eq. 2）；ARC-AGI-3 官方 RHAE 与 win rate；EdgeBench 12 小时内 best-so-far 分数及 2h/4h/…/12h 时间曲线。 | App. A.1 "Evaluation Metrics"；Table 4；Table 11；Table 5；Table 15 |
| 控制条件 | 所有方法同一初始 harness h_1、同一预算 K、AHE 每 harness 每任务 m=1；ARC-AGI-3 每后端内同一初始 prompt、工具、模型、reasoning 设定，2,000 动作上限；EdgeBench 12 小时 wall-clock 匹配，harness 编辑时间计入预算。 | §4.1；App. B.1；App. C.1 "Cost" |
| 过程统计量 | ARC-AGI-3 每局 harness 操作计数（Table 12：read/write_guide、read/write_working、read/write_skill、write_tool、run_tool、write_hooks）；案例研究中的逐关动作数（Table 13）与写入资产时间线；App. A.2 定性描述每轮修改内容与被保留/回滚的改动。 | App. B.2；Table 12；App. B.3；Table 13；App. A.2 |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索/分析循环：Harness Evolution——每轮在当前 harness h_k 下对 n 个任务各采 m 条 rollout，存入经验库 C_k；meta agent 读 Φ(C_{k−1}) 产出 h_k（k=2…K）；最终无单测取 h_K，有单测取 R̄ 最高的 harness；再对目标任务采样。Harness Scaling——每个任务从 h_1 采一条轨迹，meta agent 根据 Φ(h_{k−1}, y_{k−1}[, R]) 改 harness，再重采样，共 K 轮。 | §3.4；§3.5 |
| 关键设计：统一预算视角（明确每种方法的反馈、预算分配、改轨迹还是改 harness）；关闭 AHE 的 explore agent 以排除外部检索的基准特定 harness；Agent Debugger 把轨迹组织成文件环境并输出逐任务分析报告；§4.4 把搜索集与评测集分离。 | §3；App. A.1；§4.4 |
| 接受/选择规则：Parallel Sampling 无单测用 self judge 取 argmax，有单测取任一通过的轨迹；Sequential Refinement 无单测取 y_K，有单测取任一通过的轨迹；Harness Evolution 见上；Harness Scaling 无单测取 y_K，有单测取任一通过的轨迹。EdgeBench：meta agent 的改动需通过 validation 才安装，否则沿用旧 harness。 | §3.2–§3.5；App. C.1 |
| 规模：Terminal-Bench 89 任务 × 3 模型 × 4 方法 × 2 设定，K=5（消融 K=10）；ARC-AGI-3 25 游戏 × 2 模型 × 4 配置，每局 2,000 动作；EdgeBench 5 任务 × 2 模型 × 2 方法，每次 12 小时。 | §4.1；App. A.3；App. B.1；App. C.1 |
| 评估设计：三个设定——无单测（§4.2）、有单测（§4.3，同时报 pass@1 与 pass@5）、不相交搜索/评测任务（§4.4，仅 Harness Evolution 适用）；长程游戏上只比较两种任务级算法（Sequential Refinement、Harness Scaling），报整段运行中的最高分。 | §4；§5.1 |
| 统计显著性检验：未说明（无检验、无置信区间；Table 3 / 9 标题用"no significant gains"但无检验；Table 8 标题称差距"在 run-to-run 方差之内"但未报方差）。 | Table 3；Table 8；Table 9 |
| 重复次数 / 随机种子数：所有 Terminal-Bench 与 EdgeBench 结果为 2 次独立运行均值；ARC-AGI-3 的重复次数：未说明；随机种子：未说明。 | §4.1；App. A.3；Table 5 |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 无单测时，自动 harness evolution 不及 test-time scaling：Parallel Sampling 平均 68.2→72.3 且三模型均提升；Sequential Refinement 仅 +1.1；Harness Evolution 平均 67.4 低于 direct sampling 68.2，GPT-5.4 上 75.3→69.7；Harness Scaling 71.8，在 Claude Opus 4.6 上取得单项最佳 76.0。作者解释：自生成反馈噪声大，顺序修订会累积早期错误。 | Table 1；§4.2 |
| 有单测时仍不及：pass@1 上 Parallel Sampling 平均 82.0 最佳，Harness Evolution 69.7 仅略高于 direct 67.6；pass@5 上 Sequential Refinement 87.1 最佳，高于 Harness Evolution 79.0 与 Harness Scaling 85.6。harness 修改的收益只在可从多条轨迹中选择时出现，说明其收益主要来自多次尝试而非更好的 harness。 | Table 2；§4.3 |
| 不相交搜索/评测任务时 Harness Evolution 不泛化：Claude Opus 4.6 +1.2、GPT-5.4 +0.0、GPT-5.4 mini −1.4，均值 +0.0；作者判断演化出的修改是任务特定捷径，弱模型更易过拟合。 | Table 3；§4.4 |
| 预算翻倍到 K=10 结论不变：无单测 Harness Evolution 71.9 vs Parallel Sampling 73.6；有单测 pass@1 Harness Evolution 73.6 vs Sequential Refinement 89.6 vs Harness Scaling 90.5；held-out 上 Harness Evolution 63.3→63.0（−0.3）。 | Tables 7–9；App. A.3 |
| 对 Terminal-Bench 结果的解释：模型可能不够强或训练期已针对该基准优化；现有分数已高，剩余失败多来自模型而非 harness；Terminal-Bench 对 harness 设计不敏感。App. A.2 进一步指出编辑"记忆修复而非提炼策略"，困难失败核心不受影响，且持久 prompt 文本的增长带来上下文膨胀。 | §5；App. A.2 |
| 长程游戏是 harness evolution 的有利场景（三条件：足够难有 headroom、依赖对分布外动态的适应、设计上提供细粒度可验证反馈）。ARC-AGI-3：Harness Scaling [full harness] 相对 [VISTA*] 由 43.3→77.5（Opus 4.8，+79.0%）、21.1→38.2（GPT-5.6 Terra，+81.0%）；[VISTA]→[full harness] 为 59.3→77.5 与 25.9→38.2。 | §5；Table 4；§5.2 |
| EdgeBench：Harness Scaling 相对 Sequential Refinement 平均 48.3→54.7（Opus 4.8，+13.3%）、45.9→49.8（GPT-5.4，+8.5%）；5 任务中 GPT-5.4 有 2 个退步（nethack、openttd）、Opus 4.8 有 1 个退步（wesnoth）。记忆仍是主要修改界面；meta agent 擅长单路径深挖、有时忽略广度。 | Table 5；§5.3 |
| ARC-AGI-3 上工具是最主要的 harness 产物：每局写约 11（Opus 4.8）/ 20（GPT-5.6 Terra）个工具、调用 132 / 197 次；工具操作占 169 中的 143、260 中的 217；开放全 harness 后记忆操作减少 17% / 24%；hooks 几乎不写（0.2 / 0.0）。案例：tr87 上 GPT-5.6 Terra 由 11.8→100 RHAE，[VISTA] 1,415 动作只过 1 关，[full harness] 256 动作通关；lp85 上 Opus 4.8 第 8 关 133→20 动作；ls20 上 Opus 4.8 是唯一通关配置（666 动作，人类基线 776）。 | App. B.2；Table 12；App. B.3；Table 13 |
| 总体呼吁：评测应使用匹配预算的 test-time scaling 基线和 held-out 评测，把优化反馈与最终测量分开；现有 harness evolution 证据应谨慎解读。 | Abstract；§6 |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 自动 harness evolution 有效的场景需满足三条件：任务足够难、留有 headroom；依赖对分布外动态的适应（专用工具、skills、workflow 对成功至关重要）；环境提供细粒度、可验证的反馈。 | §5；Abstract |
| 需要可靠的外部正确性信号来约束 harness 修订；仅靠 agent 自评会伤害强模型（GPT-5.4 75.3→69.7）。 | §4.2；Table 1 |
| 评测协议要求：与 Parallel Sampling / Sequential Refinement 在匹配反馈与推理预算下比较；同时报 pass@1 与 pass@k；搜索任务与评测任务分离。 | §1；§4.3；§4.4；§6 |
| 长程游戏上应从成熟 harness（VISTA*、Claude Code CLI、Codex CLI）起步，并让 agent 能写工具（而不只是记忆）。 | §5.1；§5.2；App. B.2 |
| 所需算力/费用：Terminal-Bench 每任务每方法 K=5 条 rollout（消融 K=10）、2 次独立运行、每 rollout 独立 E2B 沙盒、单轮最大生成 128k tokens；ARC-AGI-3 每局 2,000 动作；EdgeBench 每次运行 12 小时、工作容器 4 CPU / 16 GB、judge 4 CPU / 8 GB。token 总量与美元费用：未说明；Terminal-Bench 上 harness 修改成本未计入。 | §4.1；Table 6；App. B.1；Table 14；App. A.1 "Cost" |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| 作者自认：Terminal-Bench 上的负面结果可能源于模型能力不足、训练期对该基准的优化、分数饱和或基准对 harness 不敏感，并未严格区分。 | §5 |
| 作者自认：Terminal-Bench 上不计 harness 修改成本，预算匹配实际上偏向 harness 方法（作者认为计入只会强化其负面结论；但对游戏上的正面结论未讨论此影响）。 | App. A.1 "Cost" |
| 作者自认：Table 12 的操作计数衡量使用频率而非价值；EdgeBench 上 meta agent 有时忽略广度探索。 | App. B.2；§5.3 |
| 作者自认：Terminal-Bench 的结论不外推——长程游戏上 Harness Scaling 优于 Sequential Refinement；但该结论只覆盖逐任务的 harness scaling，不涉及可复用 harness 的泛化。 | §5；§5.1 |
| 统计显著性/方差：所有结果仅 2 次独立运行取均值，未报方差、置信区间或检验；Table 3 中 ±1.2 / −1.4 的差异、Table 8 标题"在 run-to-run 方差之内"均无数值支持；ARC-AGI-3 重复次数未说明。 | §4.1；Table 3；Table 8；App. B.1 |
| 核对出的内部不一致：同一"direct sampling using initial harness"设定在 Table 1 与 Table 2 数字不同（GPT-5.4 75.3 vs 75.9；GPT-5.4 mini 59.4 vs 56.9），Table 7 与 Table 8 的 K=10 direct sampling 也不同（69.5 vs 68.5），应是不同批次运行，侧面反映 run 间噪声约 0.5–2.5 分。 | Table 1；Table 2；Table 7；Table 8 |
| 核对出的表述差异：摘要称 ARC-AGI-3 "improves over the search baseline by 80%"、EdgeBench "11%"，正文分别为 +79.0% / +81.0%（相对 [VISTA*]）和 +13.3% / +8.5%（相对 Sequential Refinement），摘要数字为两模型相对增益的约略平均；§1 则称对比对象为 "fixed-harness sequential refinement"。 | Abstract；§1；§5.2；§5.3 |
| 未评估：跨模型、跨基准、跨 harness 实现的 harness 迁移；从成熟 harness 起步的 Terminal-Bench 实验；删除 / 回滚类编辑的量化；比 GPT-5.4 mini 更小的模型。 | Table 6；§4.4；§5.1；App. A.2 |
