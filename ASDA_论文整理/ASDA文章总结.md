# ASDA 文章总结

> 论文：*ASDA: Automated Skill Distillation and Adaptation for Financial Reasoning*（Tik Yu Yim, Wenting Tan, Sum Yee Chan, Tak-Wah Lam, Siu Ming Yiu；The University of Hong Kong；arXiv:2603.16112，2026 年 3 月）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：在只有黑盒 API、不能改权重的条件下，如何把 LLM 适配到金融推理这种需要多步定量计算与领域判断的任务；现有训练无关方法（GEPA、ACE）优化的是扁平文本，在 FAMMA 上只有边际增益。 | Abstract；§1 |
| 子问题：（1）增益来自教师的更强知识还是训练数据本身（self-teaching 消融）；（2）为一个模型蒸馏的 skill 能否用于另一模型（cross-transfer）；（3）迭代精炼几轮最优。 | §5.3；§5.4；§5.1 |
| 形式化目标：未给出目标函数；流程以 Algorithm 1 描述——每轮评估得 Q+ / Q− / Q_gap，按文件归因，coverage 精炼通过 τ_cov 门，安全精炼通过 τ_safe 门，输出 K_T。 | §3.2；Alg. 1 |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被优化对象：skill 库——按 (subfield, error_type) 组织的 Markdown skill 文件（含 pattern：知识缺口描述、何时使用、分步过程、worked example / 代码模板）加 SKILL.md 导航表。 | §3.1；Fig. 2 |
| 方法本身：ASDA——教师–学生架构；Phase 1 warm-up：教师对学生错题做结构化失败标注并按 (subfield, error_type) 聚类合成 K0；Phase 2 迭代精炼：证据收集与归因 → coverage 精炼 → 安全精炼，每次更新过 verification gate；推理期 selector 读 SKILL.md 注入相关文件。 | §3；Fig. 1；§3.1；§3.2 |
| 任务领域：金融推理 QA（FAMMA-Basic-Txt），分算术（PoT 代码执行）与非算术两类。 | §4.1；§4.3 |
| 底层模型（学生）：Claude Haiku 3.5、Claude Haiku 4.5。 | §5.1；Table 3 |
| 优化器/判官/专家模型：教师 Sonnet 4.5（所有主实验）；self-teaching 消融中学生兼教师；开放题判官 Qwen-Max；MC 选择步 Qwen-Turbo；selector 为 LLM-based（具体模型在脚注 4，未说明）。 | Table 3；Table 4；§4.5；§3.1 |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 金融推理 | FAMMA-Basic（1,945 题，教材与职业考试，8 个子领域，3 个难度级）的 FAMMA-Basic-Txt 版本（OCR 抽取的文本上下文）；限制为 1,378 道英文题；FAMMA 无官方切分，作者按难度与题型（MC / 开放）分层 60/40 自切：算术 448 训练 / 300 测试，非算术 378 训练 / 252 测试；共享上下文传播到各子题。去污染：未说明。 | §4.1；§4.2；§4.3 |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | skill 库内容：文件数、每文件的 pattern、触发条件、过程、示例；安全精炼时移除或收窄指引。 | §3.1；§3.2 |
| 对比条件 | Baseline / GEPA / ACE / ASDA WU / ASDA E2（Haiku 3.5）；Baseline / ASDA WU / ASDA E2（Haiku 4.5）。 | Table 3 |
| 消融条件 | self-teaching（学生 = 教师，Haiku 3.5）vs Sonnet 4.5 教师；cross-transfer（H3.5 skill → H4.5）vs own skills；精炼轮数 WU / E1 / E2 / E3。 | Table 4；Table 5；§5.1 |
| 结果指标 | 测试集准确率（%）与相对自身 baseline 的绝对 Δ（pp）；按题型拆分的 MC Δ 与 Open Δ。 | Table 3；Table 5；§5.1 |
| 控制条件 | 所有推理 temperature 0；Qwen-Max 判官与 Qwen-Turbo MC 选择步在所有条件下固定；MC 题规则精确匹配；判官换成 GPT-4o 后 1,104 次判断一致率 99.3%（算术 0.00 pp、非算术 −0.39 pp）。 | §4.3；§4.5 |
| 过程统计量 | 教师 / 学生 token 与费用（Table 2）；每配置 10–30 个 skill 文件；一个 skill 文件额外修复 7 道相关固定收益题；Q+ / Q− / Q_gap 的规模：未说明。 | Table 2；§6；Fig. 3 |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索/分析循环：warm-up：学生做训练集 → 教师对每道错题输出 {subfield, error_type, root_cause} → 按 (subfield, error_type) 聚类 → 每簇合成一个 skill 文件 + SKILL.md；每轮精炼：每道训练题有/无 skill 各评一次 → 分 Q+ / Q− / Q_gap → 教师归因到单个文件 → 对有 Q_gap 的文件提扩展，过 τ_cov 门 → post-coverage 复验得 Q̃+ / Q̃− → 对有 Q̃− 的文件提修复，过 τ_safe 门 → 下一轮。 | §3.1；§3.2；Alg. 1 |
| 关键设计：失败驱动的结构化标注（root_cause 写知识缺口）；两级库组织与 SKILL.md 导航；推理期 selector 可选多文件；配对的有/无 skill 评估把修复与回归分开；按文件归因；coverage 与 safety 双阶段精炼平衡覆盖与回归。 | §3.1；§3.2；§2.3 |
| 接受/选择规则：候选 skill 更新须让学生重解目标题的准确率 ≥ τ（coverage：Q_gap 恢复率 > τ_cov；safety：正例准确率不低于保留阈值且负例改善，τ_safe）；不达标则教师重生成，最多 N_max 次后回退旧版。τ、N_max 的取值：未说明。 | §3.2；Alg. 1 |
| 规模：2 个学生 × 2 个任务类型；最佳操作点 2 轮精炼（E3 回退）；每配置 10–30 个 skill 文件；训练集 448 / 378 题，测试集 300 / 252 题。 | §5.1；§6；§4.2 |
| 评估设计：自建分层 held-out 测试集；逐题独立评估；算术用 PoT + MC 映射；MC 精确匹配、开放题 LLM 判官；GEPA / ACE 按各自架构适配训练条件（脚注 9，未说明细节）。 | §4.2–§4.4；Table 1 |
| 统计显著性检验 | 未说明 |
| 重复次数 / 随机种子数 | 未说明（仅称 temperature 0 以保证可复现） |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 1. Haiku 3.5 算术：warm-up +8.67 pp（49.67%），两轮精炼后 +17.33 pp（58.33%）；非算术 +2.78 / +5.95 pp（51.98% / 55.16%）。 | Table 3；§5.1 |
| 2. Haiku 4.5 尽管 baseline 更高（64.67%），算术仍 +5.99 pp（E2 70.66%）；非算术 warm-up −0.79 pp，精炼后才 +1.60 pp。 | Table 3；§5.1 |
| 3. GEPA 与 ACE 在 FAMMA 上只有边际增益：算术 +1.33 / +3.30 pp，非算术 +1.58 / +0.39 pp。 | Table 3；§5.1 |
| 4. 精炼轮数：算术 49.67 → E1 54.67 → E2 58.33 → E3 54.33；非算术 E2 55.16 → E3 51.98；两轮为最优操作点，第三轮过拟合训练集残余模式。 | §5.1 |
| 5. MC 题增益大于开放题：Haiku 3.5 算术 warm-up MC +14.39 vs Open +3.73 pp；Haiku 4.5 MC +7.91 vs Open +2.48 pp。 | §5.1 |
| 6. 回归主因是 skill-induced over-reasoning：selector 整包加载子领域全部文件，模型对不需要的题也照指引过度推理、推翻原本正确的判断。 | §5.1 |
| 7. self-teaching（Haiku 3.5 自教）算术 +6.33 pp，为 Sonnet 4.5 教师增益 +8.67 的 73%，差 2.34 pp；非算术 +1.58 vs +2.78 pp——主要增益来自训练数据而非教师知识。 | Table 4；§5.3 |
| 8. cross-transfer：Haiku 3.5 的 skill 用于 Haiku 4.5 算术净 −2.33 pp（开放题 −6.21、MC +2.16），而自身 skill +5.00 pp；结论：skill 是模型专属失败补救，应为每个部署模型独立生成。 | Table 5；§5.4；§6 |
| 9. 成本：学生专属 warm-up 蒸馏不到 $12（算术 $11、非算术 $8），两轮精炼另加 $89–135，按模型重生成在部署规模上可行。 | Table 2；§5.4；§6 |
| 10. 增益在失败模式聚类清晰、任务有明确过程结构（如 PoT 算术）时最大；错误分散（非算术）时信号弱、回归风险高。 | §6 Discussion |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 前提：有带标注答案的领域数据集（只需题目与正确答案，不需 worked solutions）；黑盒 API 即可，不需权重、梯度或执行轨迹。 | §1；§2.3；§5.3 |
| 前提：失败能按 (subfield, error_type) 清晰聚类、任务有可编码的过程结构；适合政策与程序化的工作流（如信贷评估），不适合依赖隐性案例判断的任务（如欺诈调查）。 | §6 Discussion |
| 需要：教师模型（Sonnet 4.5；或学生自教，可保留 73% 的算术增益）、推理期 LLM selector、判官（开放题）；skill 须为每个部署模型单独生成，不应跨代复用。 | Table 4；§3.1；§5.4 |
| 所需算力/费用：warm-up < $12（Haiku 3.5 学生 + Sonnet 4.5 教师计价）；两轮精炼 $89–135；超过两轮无益。Haiku 4.5 的费用：未说明。 | Table 2；§5.4；§5.1 |
| 作者建议：两轮精炼为操作点；pattern 级选择与低价值 skill 剪枝可减少回归；跨领域与跨家族验证在进行中。 | §5.1；§6 Future Work |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| （作者写到）单基准（FAMMA）、单模型家族（Claude 同时作学生与教师），不能确定分类法、skill 格式、精炼动态能否迁移到其他领域或架构。 | §6 Limitations |
| （作者）FAMMA 的 OCR 文本可能让 skill 编码伪影启发式，报告的回归率可能高估干净数据上的情况。 | §6 Limitations |
| （作者）纯基准评估，生产金融流程中的实际价值未验证。 | §6 Limitations |
| （作者）selector 整包加载子领域全部文件导致不必要指引与回归；未做 pattern 级选择与剪枝。 | §5.1；§6 Future Work |
| （作者）结果在自建切分上，不可与 Xue et al. 原报告直接比较；GEPA / ACE 此前未在 FAMMA 上评估。 | §4.2；§4.4 |

| Table 5 的 "own skills" 行（69.67，+5.00）对应 Table 3 的 Haiku 4.5 warm-up 库，而非最佳 E2 库（70.66）；cross-transfer 行所用的 Haiku 3.5 skill 库是 warm-up 还是 E2：未说明。 | Table 3；Table 5 |



