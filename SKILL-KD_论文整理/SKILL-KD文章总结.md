# SKILL-KD 文章总结

> 论文：*SKILL-KD: Contrastive Skill Distillation for LLM Agents*（Qiming Shi, Yibo Dou, Jiawen Zhu, Yulong Tao, Linbo Jin, Zhaolu Kang, Yunfan Zhou, Di Weng；arXiv:2607.28048，2026-07）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：如何在不更新权重的前提下，把更强 teacher agent 的程序性知识迁移给冻结的较弱 student agent。现有方法从 agent 自身经验或成功示范抽取 skill，但失败轨迹看不出缺了什么，成功轨迹又可能过于隐含、与 student 不匹配。 | Abstract, §1 |
| 子问题 1：如何把 teacher-student 的行为差异翻译成 student 能遵循的文本 skill patch（一次性总结不保证在 student 策略下有效）。 | §1, §3.2 |
| 子问题 2：反复局部更新引起 skill drift（过拟合个案、冗余、覆盖旧知识），如何保持库紧凑稳定。 | §1, §3.3 |
| 形式化目标：优化外部 skill 库 K 使冻结 student 在 D_train 上的任务表现提升；每条 skill p = (title, content, why, trace)；评测器 r(x, τ) ∈ [0,1]。 | §3.1 |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被优化对象：benchmark 专属的 skill 库 K（结构化 Markdown 文件，student 与 teacher 共享，整文件进 prompt）。 | §3.1, App. A |
| 方法本身：SKILL-KD = Contrastive Skill Distillation（同任务 student 失败轨迹 vs teacher 轨迹）+ Adaptive Skill Distillation（patch 经 student 重跑验证，最多 n 轮）+ Drift-Aware Skill Consolidation（多轮 agent，带 trace-linked 编辑历史，决定 add/modify/delete/skip）。 | §3.1–3.3 |
| 任务领域：搜索问答（SearchQA）、电子表格操作（SpreadsheetBench）、多模态文档问答（DocVQA）、数学推理（LiveMath）、具身环境交互（ALFWorld）。 | §4, §4.1 |
| 底层模型（student）：Group 1 Qwen3.5-4B；Group 2 Qwen3.6-35B-A3B；均冻结，本地 4×A100 80GB 部署，默认 medium reasoning effort。 | §4 Setting, App. A |
| 优化器/判官/专家模型：teacher 兼 consolidation agent：Group 1 Qwen3.7-plus，Group 2 ChatGPT-5.5（官方 API）。判官为各 benchmark 原生评测器。 | §4 Setting, App. A |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 具身交互 | ALFWorld：train 39 / val 18 / test 134；test 为 unseen 环境。 | Table 6, §4.1 |
| 电子表格 | SpreadsheetBench：train 80 / val 40 / test 280。 | Table 6 |
| 搜索问答 | SearchQA：train 400 / val 200 / test 1400。 | Table 6 |
| 数学 | LiveMath（LiveMathematicianBench）：train 35 / val 18 / test 124。 | Table 6, References |
| 文档问答 | DocVQA：train 107 / val 53 / test 374。 | Table 6 |
| 切分规则 | 有官方划分的用原生划分；无官方划分的按 2:1:7 train/val/test 确定性切分，split seed 42（沿用 SkillOpt 协议）。哪些 benchmark 用原生划分：未说明。去污染：未说明。 | §4 Setting, App. A |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | skill 库 K 中的规则（title/content 可见；why/trace 内部）；每条 patch 的 op ∈ {add, modify/update, delete, skip}。 | §3.1, §3.3, App. B |
| 对比条件 | No Skill、EvoSkill、Trace2Skill、SkillGen、SkillOpt；两组 teacher-student 配置。 | Table 1, §4 Baselines |
| 消融条件 | 信号来源变体：Student-only、Teacher-only、Batch（四条轨迹一次更新）、Pairwise（单次对比、无 adaptive）；组件消融：w/o Teacher（用 student 自身失败轨迹替代 teacher 轨迹）、w/o Adaptive、w/o Consolidation（去掉编辑历史访问）。 | §5.1, §5.2, Table 2, Table 3 |
| 结果指标 | 各 benchmark 原生 hard-success / exact-match（百分比），五个 benchmark 宏平均 Avg；Avg. Gain 相对 No Skill。 | §4, Table 1, Table 2 |
| 控制条件 | 均为 Group 1（Qwen3.5-4B / Qwen3.7-plus）做分析与消融；固定 test 划分；n = 3；每 benchmark 独立库；teacher/student 共享同一库。 | §5, App. A |
| 过程统计量 | 最终库规模（Total Rules / Total Words；按 benchmark 的规则数）；训练集按轮累计成功率；teacher 在 student 失败实例上的成功率；patch 接受率（按 teacher 成功/失败条件）；来自 teacher 失败的接受 patch 占比。 | Table 2, Table 4, Table 5, Table 7, Fig. 3 |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索/分析循环：对每个训练实例，student 带当前 K 跑；成功则跳过；失败则 teacher 在同实例跑；consolidation agent 由 (τ_S0, r_S0)、(τ_T, r_T)、K、H 生成 patch p1；student 带 Apply(p1, K) 重跑；仍失败则加入累计证据 E_i 生成 p_{i+1}（相对原始 K，取代先前 patch），直到成功或达到 n 轮。 | §3.1, §3.2 |
| 关键设计：(1) 同任务对比作为蒸馏信号，teacher 不必成功；(2) 以 student 重跑作为行为验证；(3) consolidation agent 是多轮工具调用 agent，拥有 trace-link（取回历史轨迹）与 patch 两个工具，先查相关历史再决定 add/modify/delete/skip；prompt 要求 content 为 "When <trigger>, do <action>"，触发条件须为类型级，不绑定文件名、单元格地址、房间名。 | §3.1, §3.3, App. B |
| 接受/选择规则：只有 student 重跑 r = 1 的 patch 才提交到 K；中间失败 patch 不写入 K 或 H；n 轮内未成功则无更新。 | §3.2 |
| 规模：5 个 benchmark × 2 组；训练实例共 661（39+80+400+35+107）；n = 3；submit_ops 单次最多 3 个 op。 | Table 6, App. A, App. B |
| 评估设计：固定 test 划分上报告原生指标与宏平均；ALFWorld test 为 unseen 环境作为 benchmark 内泛化检查；分析/消融只在 Group 1。 | §4, §4.1, §5 |
| 统计显著性检验 | 未说明。 |
| 重复次数 / 随机种子数 | 每个报告分数为固定 test 划分上的单次完整评测；split seed 42。 | App. A |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 主结果：Group 1（Qwen3.5-4B）宏平均 No Skill 43.5 / EvoSkill 51.5 / Trace2Skill 53.6 / SkillGen 52.5 / SkillOpt 63.5 / SKILL-KD 66.8；Group 2（Qwen3.6-35B-A3B）57.9 / 61.9 / 59.9 / 61.0 / 68.6 / 74.6。 | Table 1 |
| SKILL-KD 在两组所有五个 benchmark 上都高于 No Skill；Group 1 上 SpreadsheetBench（9.3→24.3）、LiveMath（22.4→54.8）、ALFWorld（30.6→86.6）提升最明显。 | §4.1, Table 1 |
| ALFWorld unseen 环境：Group 1 30.6→86.6，Group 2 59.7→96.3，作者认为学到的是可复用交互策略而非记忆训练环境。 | §4.1 |
| 信号来源变体（Group 1）：Student-only 60.1（96 rules, 6,821 words）、Teacher-only 58.3（最弱学习变体）、Batch 59.4、Pairwise 59.0（85 rules, 6,966 words）、SKILL-KD 66.8（38 rules, 3,010 words）。 | §5.1, Table 2 |
| 消融（Group 1）：Full 66.8；w/o Teacher 62.1；w/o Adaptive 59.0；w/o Consolidation 58.7。w/o Teacher 主要伤 ALFWorld（86.6→73.1）和 LiveMath（54.8→48.4），SearchQA（79.2→78.6）与 DocVQA（89.3→89.0）几乎不变。w/o Consolidation 对 SpreadsheetBench（24.3→14.6）、LiveMath（54.8→27.4）伤害最大。 | §5.2, Table 3 |
| 库规模：无 consolidation 85 rules（SpreadsheetBench 50 rules, 2,930 words）；有 consolidation 38 rules（3,010 words；SpreadsheetBench 6 rules, 522 words），且分数更高（66.8 vs 58.7）。 | §5.3, §5.4, Table 4 |
| adaptive 轮数：训练集累计成功率 58.9% → 67.7%（第 1 轮）→ 70.8% → 72.7%（n=3）；第一轮贡献最大，后续增益集中在 SpreadsheetBench、LiveMath、ALFWorld。 | §5.5, Table 7, Fig. 3 |
| teacher 噪声：在 student 失败实例上 teacher 平均只成功 46.1%；接受率 teacher 成功时 45.3% vs 失败时 30.2%；平均 39.2% 的被接受 patch 来自 teacher 失败轨迹；LiveMath 中 teacher 成功率 10.0%，Accept|T success 0.0，100.0% 的接受 patch 来自 teacher 失败。 | §5.6, Table 5 |
| 三类 drift 的定性案例：DocVQA 数值单位规则边界反复漂移（case-specific）；SpreadsheetBench 绑定 P6:Q6、R3:R4、C4:F6 的坐标级修复（skill-bloat）；通用 openpyxl 指令覆盖分层奖金算法（destructive-update）。 | §5.4 |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 需要显式成功指标的任务（r(x, τ) 可自动判定）；开放式任务需人工/偏好/领域反馈，未覆盖。 | Limitations |
| 需要一个更强的 teacher 且能在同一实例、同一 skill 库上运行；teacher 不必完美，但轨迹要能揭示可复用修正而非相同失败模式。 | §3.1, §5.6, Limitations |
| 迁移内容限于 student 能通过 prompt 遵循的程序性指导；超出自然语言指令遵循的能力需结合检索、工具学习或模型适配。 | Limitations |
| 每个 benchmark 需要训练划分（本文 35–400 条）；库按 benchmark 单独训练，整文件进 prompt（不含检索/路由）。 | Table 6, App. A |
| 所需算力/费用：student 本地 4×A100 80GB；teacher 与 consolidation agent 走 API；token / 费用 / 时间未说明。 | App. A |
| 作者建议：对生成的 skill 做人工审查并保留人工监督；涉及敏感数据时轨迹日志与 skill 需匿名化。 | Ethical Considerations |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| 作者自认：只在显式指标 benchmark 上评估；依赖信息量足够的 teacher-student 对比；只覆盖可由 prompt 传达的程序性知识。 | Limitations |
| 统计显著性/方差：每个分数为单次完整评测，无重复、无 std、无检验；Table 2 中 Pairwise 59.0、Batch 59.4、Student-only 60.1 的差异以及 Group 2 SearchQA（SkillOpt 80.3 vs SKILL-KD 80.6）、DocVQA（91.4 vs 92.2）的差距在无方差下无法判定。 | App. A, Table 1, Table 2 |
| 成本未报告：无 token、费用、wall-clock、consolidation agent 调用轮数；无法与 baseline 做预算匹配比较。 | App. A（只给硬件与 API 说明） |
| 内部核对：Student-only（Table 2，60.1）与 w/o Teacher（Table 3，62.1）都只用 student 失败轨迹；二者差别（Student-only 无 adaptive 重跑、w/o Teacher 保留重跑）只由 §5.1 "without behavioral validation" 与 §5.2 "iterative refinement alone" 的措辞暗示，正文未明确定义，2.0 的差距无法归因。 | §5.1, §5.2, Table 2, Table 3 |
| 内部核对："homogeneous vs heterogeneous" 结论来自两组不同 student、不同 teacher、不同分数的比较，不是受控实验。 | §4.1 |
| 内部核对：ALFWorld 最终库仅 4 条规则，各对应一类任务模板（看灯、清洗、冷却、加热后放置）；"from empty library" 只由 No Skill = 0 rules 暗示，未明说。 | App. C, Table 2 |
| 无回归检查：patch 只在当前实例上验证，未检查对已解决实例的影响；无 held-out 验证集用于接受 patch。 | §3.2, §4 Setting |
| 未测项：同一库跨 student/跨 benchmark 迁移、弱→强方向、与成熟 harness 结合、哪些 benchmark 使用原生划分。 | §4 Setting, App. A |
