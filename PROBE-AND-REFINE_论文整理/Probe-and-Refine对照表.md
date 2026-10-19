# PROBE-AND-REFINE 对照表

论文：*Probe-and-Refine Tuning of Repository Guidance for Coding Agents*（Asa Shepard, Jeannie Albrecht；Williams College；arXiv:2606.20512，2026 年 6 月）

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 部署前离线调参：每个仓库在单一 pinned commit 上一次性生成 static_kb，再经 3–5 轮 probe-and-refine 精炼成 guidance 文本；生成后固定，复用于该仓库所有后续 issue。调参阶段不跑 agent loop、不用工具。 | §3.3, §4.2, Fig 1 |
| 适用环境 | SWE-bench Verified 全部 500 实例、12 个仓库（Django 231 个，占 46%）；固定的 ReAct-style bash coding agent；主实验 200 步，另扫 25/50/100 步；有效上下文硬截断 16k tokens，每轮最多 2048 输出 tokens，fallback 512 tokens，命令输出截断约 3000 字符。 | §3.1, §3.4, §4.2, Table 3 |
| 轨迹来源 | 调参阶段没有 agent 轨迹：由与消费模型相同的模型（Qwen3.5-35B-A3B；§7 为 Nemotron）从仓库代码合成 probe（每轮 10 个，温度 0.9），对每个 probe 做单次 patch 尝试。评测轨迹来自 coding agent 在 SWE-bench Verified 上的运行，只用于评估与分析，不回流到调参。 | §3.3, §3.4, Fig 1 |
| 可见轨迹 | 调参时判官只看单次尝试的 patch 与 expected behaviors（评为 strong/partial/missing）。分析阶段作者看评测运行的 resolve/coverage/precision、fallback 率、patch 产出步数分布、每实例步数、撞步数上限数、patch-apply 失败数、no-command prose 事件数、repeated-command stall 实例数。 | §3.3, §4.4, Fig 5, Table 12 |
| 交给优化器的输入 | 步骤 2（每 probe）：仓库上下文 + 当前 guidance + probe；步骤 3（每 probe）：尝试的解 + expected behaviors，输出评判与 per-probe 编辑；步骤 4：所有 probe 结果聚合，输出针对特定 guidance section 的编辑。per-probe 编辑与聚合编辑合并去重后每轮最多 5 条，由确定性程序机械应用（插入/修改/强化 section、过滤 boilerplate、裁剪最长 bullet 以满足 3000 字符上限）。 | §3.3 |
| 诊断依据 | 调参：单次 LLM 判官（strong/partial/missing）+ 单次聚合诊断，均为同一模型。评测：官方 SWE-bench Verified harness，以仓库 held-out 测试套件通过与否判定 resolved。 | §3.3, §2, §3.1 |
| 关注的失败 | 调参时只泛称 "shortcomings"。内容分析把新增 104 行分为 procedural 49 行（47%，对应"不诊断就改"）、structural 31 行（30%，定位到具体文件/函数）、quality gate 24 行（23%，对应伪造测试结果、空 patch）。评测分析关注：定位到错误文件、fallback、patch-apply 失败、no-command prose、repeated-command stall、exploration-path divergence、self-sabotage。 | §3.3, Table 2, §5.3, §6.4, §7.3–7.4, Table 12 |
| 禁止使用 | 精炼过程不使用任何 SWE-bench 评测实例；probe 与评测实例不同以防污染；guidance 只从单一 pinned commit 生成。模型预训练是否含 SWE-bench 无法排除（三条件同模型，故不影响相对比较）。 | Fig 1, §3.3, §4.2, §3.1 |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 只改每仓库一个 ≤3000 字符（约 750 tokens，约占 16k 上下文 5%）的 guidance 文本（AGENTS.md 类）；scaffold、模型、评测 harness 全部固定。 | §3.1, §3.2 |
| prompt / 工具描述 | 改的是喂给 agent 的仓库 guidance 文本：泛泛规则被仓库特定规则替换，例如 "Run the smallest relevant test first" → 引用具体测试类/方法路径；12 仓库均自发收敛到 reproduce-first 工作流。工具描述未改。 | §3.3, Fig 2 |
| skill / 记忆 | 未说明（guidance 是跨 issue 复用的持久 artifact，但原文不称之为 skill/memory）。 | §3.3 |
| 控制逻辑（执行前） | 未说明（scaffold 固定，未增加动作校验/路由）。 | §3.1 |
| 控制逻辑（执行后） | 未由本方法修改。scaffold 自带：步数预算；超预算后单次 fallback 生成 patch；patch 清洗（删除测试文件改动、强制 diff 格式）——三条件相同。 | §3.4 |
| 明确不改 | 模型权重（无 RL、无梯度更新）、scaffold、评测 harness、上下文窗口与温度（除 probe 生成 0.9 外均 0.0）。 | §3.3, §3.1, §4.2 |
| 对照组 | no_context：裸 agent prompt。static_kb：两层固定知识库——tree-sitter 解析的结构层（hub、入口、import 关系）+ 一次性 LLM 生成的仓库无关通用建议层；它也是 probe_refined 的起点。 | §3.2 |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | 每仓库 3–5 轮，每轮约 22 次单次 LLM 调用（10 probes，步骤 2、3 各 10 次 + 生成与聚合各 1 次），每次约 8k 输入 / 约 2k 输出 tokens。12 仓库中 1 个 3 轮停、4 个 4 轮停、7 个跑满 5 轮（平均 4.5）；Qwen 共生成 341 个 probe、应用 175 条编辑。美元与 wall-clock 未说明。 | §3.3, Table 1, Table 10 |
| 诊断 | 含在上述每轮约 22 次调用中（每 probe 1 次判官 + 每轮 1 次聚合）；无额外成本项。 | §3.3 |
| 搜索补丁 | 编辑由确定性程序机械应用，不调用 LLM；每轮最多 5 条编辑。 | §3.3 |
| 验证 | 调参阶段无基于评测的验证，仅以"连续两轮 guidance 无变化（或去重后无 probe 存活）"早停。评测：4 trials × 500 实例 × 3 条件 @200 步，另 25/50/100 步各单 trial，Nemotron 各条件单 trial。推理开销：probe_refined 比 no_context 多约 56% prompt tokens；200 步相对 25 步，no_context 多用 2.3× tokens，probe_refined 多用 2.8×。绝对 token 数未说明。 | §3.3, §4.2, §4.3, §6.3, §7.2 |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | resolve rate（4 trials，200 步）：probe_refined 33.0±1.8% vs static_kb 28.3±1.4% vs no_context 25.5±2.2%；coverage 56.2±1.8 / 50.6±2.3 / 41.7±2.9；precision 58.6±1.6 / 56.0±2.5 / 61.0±2.6（条件间差异 p=0.119）。混合效应 OR：probe vs none 2.11（p<0.001）、static vs none 1.34（p=0.004）、probe vs static 1.58（p<0.001）。7.5 pp 总增益 = 2.8（static_kb）+ 4.7（精炼）。 | Table 4, Table 5, Table 8, §4.3, §4.4 |
| 对其他任务是否有效 | probe 为合成任务、评测为真实 SWE-bench 实例（结构上 held-out）；guidance 从单一 pinned commit 生成后用于该仓库所有实例（不同 commit）。Django 子集 35.3 vs 31.2 vs 27.7；非 Django 30.9 vs 25.8 vs 23.5。≥3/4 trials 的独占解：probe 31、none 8、static 4；easy core 65、hard core 342。 | §4.2, Table 7, Table 6 |
| 回归：流程层面 | 调参循环无非回归门控：编辑不经评测验证即机械应用；早停只看稳定性。 | §3.3 |
| 回归：结果层面 | 4 trials mean±SD，ordering probe > static > none 在 4/4 trials 成立；每 trial McNemar、Wilson 95% CI（T2 的 probe vs none CI 小幅重叠但 McNemar p<0.001）。报告的退步：50 步时 probe_refined 23.4% < static_kb 29.8%（单 trial McNemar p=0.022）；Nemotron 自调时所有 guidance 低于 no_context（28.4 > 27.0 > 24.6，单 trial）；跨模型迁移 13.2%。 | Table 4, §4.3, Table 9, §6.3, Table 11, Table 12 |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 跨家族、强→弱、guidance 冻结：Qwen 调出的 probe_refined 原样给 NVIDIA-Nemotron-3-Nano-30B-A3B（约 3.5B 激活），resolved 66（13.2%）vs Nemotron 自调 probe_refined 135（27.0%）；Nemotron no_context 为 142（28.4%）。Table 12（对照列为 Nemotron 自调）：agent-loop patches 174 vs 351；fallback 324 vs 3；平均步数 81.0 vs 39.2；撞步数上限 83 vs 11；patch-apply 失败 246 vs 14；no-command prose 事件 2,141 vs 1,252；repeated-command stall 实例 172 vs 41；agent-loop precision 37.9% vs 38.5%。再适配（Nemotron 自调）：循环退化，平均 3.8/5 轮、165 probes、78 edits，12 仓库中 5 个第一轮后零编辑；结果 27.0% 低于 no_context 28.4%。反向（Nemotron→Qwen）与同家族版本升级未测。 | §7.2–7.4, Table 10, Table 11, Table 12 |
| 换任务 | 跨仓库：每仓库单独调参，不迁移。跨 issue/commit：同仓库内直接复用于所有实例（Table 3 的 pinned commit 与实例目标 commit 不同）。 | §3.3, §4.2, Table 3 |
| 换执行环境 | 未说明（单一 scaffold、单一 benchmark）；作者指出带上下文摘要或树搜索的 scaffold 可能与 guidance 有不同交互。 | §9 |
| 换 Harness 实现 | 未测：作者明确 "hold a single scaffold fixed across all conditions"。 | §2 |
| 补充 | 作者自认限制：低预算（25/50/100 步）与 Nemotron 全部单 trial；精炼 guidance 长 63%，未做 padding/truncation 消融；只在一个能跑通循环的模型上验证，循环 prompt 按 Qwen 输出格式设计、未为 Nemotron 适配；probe 集/随机种子敏感性未测；Django 占 46%，11 个非 Django 仓库最少 1–2 实例；file-structure 剥离消融（定位假设的直接检验）未做；跨模型只测了 incidental transfer，专为跨模型设计的 guidance 未测。 | §9, §6.4, §7.3, §7.4 |
