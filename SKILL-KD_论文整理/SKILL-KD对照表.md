# SKILL-KD 对照表

论文：*SKILL-KD: Contrastive Skill Distillation for LLM Agents*（Qiming Shi, Yibo Dou, Jiawen Zhu, Yulong Tao, Linbo Jin, Zhaolu Kang, Yunfan Zhou, Di Weng；arXiv:2607.28048，2026-07）

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 离线、部署前：只在训练划分上逐实例构建 skill 库；评测时库固定，整个 skill 文件放进 prompt。 | §3.1, §4 Setting, App. A |
| 适用环境 | 五个 benchmark：SearchQA、SpreadsheetBench、DocVQA、LiveMath、ALFWorld。test 规模：SearchQA 1400、SpreadsheetBench 280、DocVQA 374、LiveMath 124、ALFWorld 134。SearchQA/DocVQA/LiveMath 为单轮问答或选择题；SpreadsheetBench 多轮代码执行（openpyxl + pandas，≤12 turns）；ALFWorld ≤50 步/episode。 | §4, Table 6, App. A |
| 轨迹来源 | 训练实例上：student 先带当前库 K 跑（τ_S0）；若失败，teacher 在同一实例、同一库上跑（τ_T）；之后 student 带 patch 重跑（τ_S1…）。两组配置：Group 1 student Qwen3.5-4B / teacher Qwen3.7-plus；Group 2 student Qwen3.6-35B-A3B / teacher ChatGPT-5.5。student 本地部署（4×A100 80GB），teacher 与 consolidation 模型走官方 API。 | §3.1, §3.2, §4 Setting, App. A |
| 可见轨迹 | consolidation agent 看到：当前 skill 列表、完整编辑日志、当前任务、student/teacher/retry 轨迹（各含 skill_used、model_output、evaluation）及每条轨迹的训练评测记录；历史轨迹不默认进上下文，通过 trace-link 工具按需取回。 | §3.3, App. B |
| 交给优化器的输入 | 第 1 轮：p1 = C(K, H, (τ_S0, r_S0), (τ_T, r_T))；第 i+1 轮另加累计证据 E_i = {(τ_S^j, p_j, r_S^j)}；每个新 patch 相对原始 K 提出，并取代该实例之前的 patch。 | §3.2 |
| 诊断依据 | 任务评测器 r(x, τ) ∈ [0,1]，r = 1 视为成功；指标为各 benchmark 原生 hard-success / exact-match。 | §3.1, §4 |
| 关注的失败 | student 初次失败（r ≠ 1）触发流程；库层面关注三类 skill drift：case-specific、skill-bloat、destructive-update。 | §3.1, §5.4 |
| 禁止使用 | SKILL-KD 只用训练划分构建 skill；验证划分只给需要验证/选择的 baseline，"is not used to train or select SKILL-KD skills"。 | §4 Setting, App. A |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 只改外挂 skill 库 K（一份结构化 Markdown 文档，student 与 teacher 共享）；student 模型权重固定。 | §3.1 |
| prompt / 工具描述 | 训练重跑和 held-out 评测时，当前 benchmark 的整个 skill 文件放进 prompt（无检索/路由）；student 只看到 title 与 content。模型本体 system prompt 是否改动：未说明。 | §3.1, App. A |
| skill / 记忆 | 每条 skill p = (title, content, why, trace)；why/trace 只存内部编辑历史 H，供 consolidation agent 用。操作集 {add, modify, delete, skip}（工具 schema 中为 add/update/delete，单次最多 3 个 op）。库按 benchmark 分开训练与评测。 | §3.1, §3.3, App. A, App. B |
| 控制逻辑（执行前） | 未说明。 | — |
| 控制逻辑（执行后） | adaptive 重跑验证：patch 应用后 student 重跑，r = 1 才提交；最多 n = 3 轮；n 轮内未成功则该实例不产生更新；中间失败 patch 不写入 K 或 H。 | §3.2, §5.5, App. A |
| 明确不改 | student 模型权重（"the student model is fixed"；"without weight updates"）。 | §1, §3.1 |
| 对照组 | No Skill、EvoSkill、Trace2Skill、SkillGen、SkillOpt（Table 1）；信号变体 Student-only、Teacher-only、Batch、Pairwise（Table 2）；消融 w/o Teacher、w/o Adaptive、w/o Consolidation（Table 3）。 | §4 Baselines, §5.1, §5.2 |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | 训练集大小：ALFWorld 39、SpreadsheetBench 80、SearchQA 400、LiveMath 35、DocVQA 107。token / 费用 / 时间：未说明。（推断：每个 student 失败实例最多 1 次 teacher 运行 + 最多 3 次 student 重跑，依据 §3.1–3.2 流程与 n = 3。） | Table 6, App. A |
| 诊断 | 由任务评测器打分；额外成本未说明。 | §3.1 |
| 搜索补丁 | consolidation agent 为多轮 agent，先多次调用 trace-link 再调用 patch 工具；轮数与费用未说明。训练集累计成功率：n=0 58.9% → n=1 67.7% → n=2 70.8% → n=3 72.7%。 | §3.3, §5.5, Table 7 |
| 验证 | 验证即当前实例上的 student 重跑；无单独验证集；每个报告分数来自固定 test 划分上的单次完整评测。 | §3.2, App. A |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | test 宏平均：Group 1 No Skill 43.5 → SKILL-KD 66.8；Group 2 57.9 → 74.6。训练集累计成功率 n=3 时 72.7%（Group 1）。 | Table 1, Table 7 |
| 对其他任务是否有效 | held-out test 划分；ALFWorld test 为 unseen 环境：Group 1 30.6 → 86.6，Group 2 59.7 → 96.3。跨 benchmark 迁移：未测（库按 benchmark 分开）。 | §4.1, App. A |
| 回归：流程层面 | 无 non-regression 门控；只靠 consolidation agent 查阅编辑历史与 trace 避免 destructive update。 | §3.3, §5.4 |
| 回归：结果层面 | 单次完整评测，无 std、无显著性检验。SKILL-KD 在两组所有 benchmark 均高于 No Skill；Group 2 SpreadsheetBench 中 EvoSkill 32.9、Trace2Skill 33.2 低于 No Skill 38.2（baseline 退步）。 | App. A, Table 1 |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 强 → 弱（teacher → student），student 冻结。两组各自独立训练一套库；同一库未迁移到其他 student。同家族（Group 1 Qwen）与跨家族（Group 2 Qwen ← ChatGPT-5.5）都有效，作者据此称不依赖架构相似；两组 student/teacher 均不同，非受控比较。 | §4 Setting, §4.1 |
| 换任务 | 每个 benchmark 单独一个库；未跨 benchmark 迁移。 | App. A |
| 换执行环境 | 未说明。 | — |
| 换 Harness 实现 | 未说明。 | — |
| 补充 | 作者自认局限：只在有显式成功指标的 benchmark 上评估；依赖信息量足够的 teacher-student 行为对比（teacher 不必完美，但不能是相同失败模式）；只适合 student 能通过 prompt 遵循的程序性指导。 | Limitations |
