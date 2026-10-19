# ASDA 对照表

论文：*ASDA: Automated Skill Distillation and Adaptation for Financial Reasoning*（Tik Yu Yim, Wenting Tan, Sum Yee Chan, Tak-Wah Lam, Siu Ming Yiu；The University of Hong Kong；arXiv:2603.16112，2026 年 3 月）

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 离线、部署前：Phase 1 warm-up 从训练集失败分析建初始 skill 库 K0；Phase 2 在训练集上迭代精炼；推理时由 selector 按题注入 skill 文件。不改权重，只需黑盒 API。 | §3；Fig. 1；§3.1 |
| 适用环境 | FAMMA-Basic-Txt 金融推理基准（教材与职业考试题，8 个金融子领域，3 个难度级）；只取 1,378 道英文题；算术题用 Program-of-Thought 代码执行 + MC 映射步（非算术题的执行方式未单独说明）；单题独立评估，非多步 agent 环境。 | §4.1–§4.3 |
| 轨迹来源 | 学生模型（Haiku 3.5 或 Haiku 4.5）在训练集上的作答：warm-up 用答错题的错误答案与推理轨迹；精炼阶段每道训练题各跑"注入当前 skill 库"与"无 skill"两次，得配对结果。 | §3.1；§3.2 |
| 可见轨迹 | 教师（Sonnet 4.5）看到：题目、学生错误答案与推理轨迹、ground-truth 答案；精炼时还看到学生推理轨迹与当时加载的 skill 文件。 | §3.1；§3.2 |
| 交给优化器的输入 | warm-up：教师对每道错题输出结构化标注 {subfield, error_type, root_cause}，按 (subfield, error_type) 聚类后每簇生成一个 skill 文件；精炼：按文件归因后的 Q+（有 skill 才对）、Q−（有 skill 反而错）、Q_gap（两者都错）证据集；安全精炼时 Q̃+ 作保留约束、Q̃− 作修复目标。 | §3.1；§3.2；Alg. 1 |
| 诊断依据 | 教师模型做失败分析，error_type 限定在预定义的十类分类法内，root_cause 要写知识缺口而非表面计算错误；正确性判定：MC 题规则精确匹配，开放题 Qwen-Max 判官；每道题由教师归因到"最该负责的单个 skill 文件"。 | §3；§3.1；§3.2；§4.3 |
| 关注的失败 | 十类 error type（具体列表在脚注 3，HTML 未渲染，未说明）；精炼阶段区分 coverage gaps（skill 尚未覆盖）与 regressions（skill 过拟合窄模式误导相近题）；抽样回归案例中最常见的是 skill-induced over-reasoning。 | §3；§3.2；§5.1 |
| 禁止使用 | 原文未列明令禁止项。说明的约束：所有结果在 held-out 测试集上报告；蒸馏只用训练集的问答对（题目与正确答案，无 worked solutions）；不用梯度、不改权重。 | §4.2；§5.3；§1 |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 改的是 skill 库（Markdown skill 文件 + SKILL.md 导航表），推理时注入学生 prompt；模型权重不改；兼容 Agent Skills 开放标准。 | §3.1；Abstract |
| prompt / 工具描述 | 学生使用标准任务 prompt，skill 文件作为"领域知识"注入其中；prompt 主体本身是否修改：未说明。 | §3.1；§4.4 |
| skill / 记忆 | skill 文件按 (subfield, error_type) 一文件（如 fixed_income/wrong_method_selection.md），另有 common/ 跨子领域文件；每个文件含多个 pattern，每个 pattern 有知识缺口描述、"何时使用"条件、分步推理过程、worked example 或代码模板；SKILL.md 汇总各 skill 范围并给出关键词→文件路径映射；每个配置生成 10–30 个 skill 文件。 | §3.1；Fig. 2；§6 Future Work |
| 控制逻辑（执行前） | 推理时 LLM-based selector 读题目与 SKILL.md 映射表，识别子领域并匹配 pattern，可为一题选多个文件；当前实现把匹配子领域的所有文件整包加载。selector 用的模型：脚注 4，未说明。 | §3.1；§5.1 |
| 控制逻辑（执行后） | 推理时无执行后校验（未说明）；离线精炼中每次 skill 更新须过 verification gate：学生重解目标题，准确率达阈值 τ（τ_cov / τ_safe）才提交，否则教师重新生成，最多 N_max 次后回退到旧版本。 | §3.2；Alg. 1 |
| 明确不改 | 模型权重（无权重更新、无梯度计算）；评估组件固定：Qwen-Max 判官、Qwen-Turbo MC 选择步、temperature 0。 | §1；§3；§4.5 |
| 对照组 | Baseline（标准任务 prompt、无 skill、无优化 prompt）；GEPA、ACE（仅 Haiku 3.5）；self-teaching（学生兼教师）；cross-transfer（Haiku 3.5 的 skill 用于 Haiku 4.5）vs own skills。 | §4.4；Tables 3–5 |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | 费用按 Haiku 3.5 学生 / Sonnet 4.5 教师计（Sonnet $3/$15、Haiku 3.5 $0.80/$4 每百万 token；Qwen 判官与 MC 选择 token 计入学生侧并按 Haiku 3.5 价保守计价，实际 DashScope 约低 10 倍；2026 年 3 月价）。算术 warm-up 学生 8.07M in / 331K out；非算术 warm-up 学生 5.80M / 194K。warm-up 费用主要来自 skill 增强评估。 | Table 2；§4.5 |
| 诊断 | 教师 token：算术 warm-up 545K in / 121K out；非算术 warm-up 485K / 94K。 | Table 2 |
| 搜索补丁 | 两轮精炼（E1–E2）：算术 教师 16.1M / 2.1M、学生 65.4M / 741K，$135；非算术 教师 9.2M / 1.2M、学生 51.2M / 598K，$89。精炼更贵是因为每轮反复的 train-and-test 评估与教师分析。 | Table 2；§4.5 |
| 验证 | warm-up 总费用：算术 $11、非算术 $8（作者概括为"学生专属 warm-up 蒸馏不到 $12，两轮精炼另加 $89–135"）。verification gate 的重解次数单独未报；Haiku 4.5 配置的费用、wall-clock、推理期 selector 的额外 token：未说明。 | Table 2；§5.4；§6 |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | 测试集准确率（%）。Haiku 3.5：算术 41.00→WU 49.67（+8.67）→E2 58.33（+17.33）；非算术 49.21→WU 51.98（+2.78）→E2 55.16（+5.95）。Haiku 4.5：算术 64.67→WU 69.67（+5.00）→E2 70.66（+5.99）；非算术 57.14→WU 56.35（−0.79）→E2 58.74（+1.60）。GEPA 算术 42.33（+1.33）/ 非算术 50.79（+1.58）；ACE 44.30（+3.30）/ 49.60（+0.39）。 | Table 3；§5.1 |
| 对其他任务是否有效 | 所有结果在自建 held-out 测试集（算术 300、非算术 252）上；跨领域（医疗、法律）未测，列为未来工作。 | §4.2；§6 Future Work |
| 回归：流程层面 | 有：每轮配对评估识别 Q−；安全精炼让教师"移除或收窄"导致回归的指引，并以 Q̃+ 为保留约束；verification gate τ_safe 要求正例准确率不低于保留阈值且负例改善；失败最多 N_max 次后回滚。观察到第 3 轮两域回退后选 E2 为操作点。 | §3.2；Alg. 1；§5.1 |
| 回归：结果层面 | 测试集上报告了负 Δ：Haiku 4.5 非算术 WU −0.79；跨模型迁移 −2.33（开放题 −6.21）；按题型拆分 MC / open Δ；抽样回归案例定性为 over-reasoning。未报告逐题 flip 计数、重复次数、方差或显著性检验；temperature 0，重复次数未说明。 | Table 3；Table 5；§5.1；§4.5 |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 同族弱→强、冻结迁移：Haiku 3.5 失败蒸馏的 skill 用于 Haiku 4.5 算术 300 题，64.67→62.33（−2.33；MC +2.16，Open −6.21）；对照 Haiku 4.5 自身 skill（warm-up 库）69.67（+5.00；MC +7.91，Open +2.48）。只测算术、只测一个方向；无再适配实验；跨家族（GPT、DeepSeek）未测。 | Table 5；§5.4；§6 |
| 换任务 | 未测：算术与非算术各自独立蒸馏；跨领域迁移列为未来工作。 | §4.2；§6 Future Work |
| 换执行环境 | 未说明。 | — |
| 换 Harness 实现 | 未说明；skill 文件声称兼容 Agent Skills 标准，但未在其他 harness 上评估。 | §3.1；Abstract |
| 补充 | 作者写到：单基准、单模型家族（Claude 同时担任学生与教师），不能确定分类法、skill 格式、精炼动态是否迁移到其他领域或架构；OCR 文本可能让 skill 编码伪影启发式，回归率可能被高估；纯基准评估，生产价值未验证；selector 整包加载导致不必要指引，未来做 pattern 级选择与低价值 skill 剪枝。 | §6 Limitations；§6 Future Work |
