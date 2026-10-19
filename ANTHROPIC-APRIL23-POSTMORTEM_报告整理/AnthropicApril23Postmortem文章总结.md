# AnthropicApril23Postmortem 文章总结

> 报告：*An update on recent Claude Code quality reports*（Anthropic Engineering；2026-04-23；https://www.anthropic.com/engineering/april-23-postmortem）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：过去一个月用户报告 Claude Code 质量下降，原因是什么、如何修复、以后怎样避免。 | 开头段 "We traced recent reports…" |
| 子问题 1：为什么内部使用与评测起初无法复现，且报告难以与反馈正常波动区分。 | 开头段 "neither our internal usage nor evals initially reproduced" |
| 子问题 2：一行为新模型（Opus 4.7）写的 system prompt 为何在原评测集上无回归却损害 coding quality。 | "A system prompt change to reduce verbosity" |
| 形式化目标：无（工程事故复盘）。 | — |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被测量对象：Claude Code（以及 Agent SDK、Cowork）harness 的三项变更：默认 reasoning effort、thinking 清除逻辑、system prompt 一行。 | 开头段三条 |
| 方法本身：事故排查 + 逐行 system prompt ablation（更广评测集）+ Code Review 回测。 | "A system prompt change…"；"A caching optimization…" |
| 任务领域：代码任务（Claude Code 的日常使用）；具体评测任务：未说明。 | — |
| 底层模型：Sonnet 4.6、Opus 4.6、Opus 4.7（4 月 16 日发布于 Claude Code）。 | 开头段；"A system prompt change…" |
| 优化器/判官/专家模型：无自动优化器；判官为内部评测集与人工；Code Review 回测用 Opus 4.7 与 Opus 4.6。 | "A caching optimization…" |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 用户反馈 | /feedback 命令提交的问题、线上发布的可复现示例；规模未说明。 | "Going forward" 末段 |
| 内部评测 | 原评测集（上线前，无回归）与事后"a broader set of evaluations"；评测名称、任务数、切分：未说明。 | "A system prompt change…" |
| 代码审查回测 | 引发 bug 的 pull request 及相关代码仓库。 | "A caching optimization…" |
| 去污染 | 未说明。 | — |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | system prompt 的各行（逐行移除）；默认 effort 档位（high / medium / xhigh）；thinking 清除标志（clear_thinking_20251015 + keep:1）。 | 各小节 |
| 对比条件 | prompt 含 vs 不含某行；Opus 4.7 vs Opus 4.6（Code Review 回测）。 | "A system prompt change…"；"A caching optimization…" |
| 消融条件 | 逐行移除 system prompt（"removing lines… to understand the impact of each line"）。 | "A system prompt change…" |
| 结果指标 | 评测分数（一项评测下降 3%，单位未说明）；延迟；token 用量；usage limit 消耗；用户报告。 | 各小节 |
| 控制条件 | 未说明（评测集、样本数、运行次数均未给出）。 | — |
| 过程统计量 | 时间线：3/4、3/26、4/7、4/10（v2.1.101）、4/16、4/20（v2.1.116）、4/23；根因确认 >1 周（thinking 清除 bug）；上线前内部测试"multiple weeks"。 | 各小节 |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索/分析循环：收集用户报告 → 内部复现（起初失败）→ 分别定位三项变更 → 回退/修 bug → 事后逐行 ablation 与 Code Review 回测。 | 全文 |
| 关键设计：ablation 用更广评测集、对每个模型分别跑；Code Review 回测时提供相关代码仓库作为上下文。 | "A system prompt change…"；"A caching optimization…" |
| 接受/选择规则：原评测集无回归即上线（事前）；发现 3% 下降即立即回退（事后）。 | "A system prompt change…" |
| 规模：三项变更；三个模型；一项评测报告了数字。 | 开头段 |
| 评估设计：内部评测集 + 用户反馈；held-out 划分：未说明。 | — |
| 统计显著性检验 | 未说明。 |
| 重复次数 / 随机种子数 | 未说明。 |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 质量下降来自三项独立的 harness 变更；API 与推理层未受影响；4 月 20 日 v2.1.116 全部解决。 | 开头段 |
| 3 月 4 日默认 effort high→medium 是错误取舍；内部评测中 medium "slightly lower intelligence" 但延迟显著更低；用户仍感到变笨；4 月 7 日回退，Opus 4.7 默认 xhigh、其他模型 high。 | "A change to Claude Code's default reasoning effort" |
| 3 月 26 日的缓存优化本应在空闲 >1 小时后清一次旧 thinking，bug 使其每轮都清并只保留最近一个 reasoning block，表现为遗忘、重复、奇怪工具选择，并因持续 cache miss 导致 usage limit 消耗快；4 月 10 日 v2.1.101 修复。 | "A caching optimization that dropped prior reasoning" |
| 该 bug 通过了多轮人工与自动代码审查、单元/端到端测试、自动验证和 dogfooding；两项无关实验（server-side message queuing；thinking 显示方式）掩盖了它；根因确认花了超过一周。 | "A caching optimization…" |
| Code Review 回测：提供完整仓库上下文时 Opus 4.7 找到了该 bug，Opus 4.6 没有。 | "A caching optimization…" |
| 为 Opus 4.7 的冗长问题加入的一行长度限制 prompt，经数周内部测试且原评测集无回归，4 月 16 日上线后"in combination with other prompt changes"损害 coding quality；更广评测集的逐行 ablation 发现一项评测在 Opus 4.6 与 4.7 上各降 3%；4 月 20 日回退。 | "A system prompt change to reduce verbosity" |
| 三项变更影响不同流量切片与时间表，叠加后看似广泛而不一致的退化，难与正常反馈波动区分。 | 开头段 |
| 后续措施：更多员工用公开版本；改进并发布 Code Review；每次 prompt 更改跑 per-model 广评测集、持续逐行 ablation、新审计工具；CLAUDE.md 要求模型特定更改门控到目标模型；可能影响智能的更改加 soak period、更广评测、灰度发布。 | "Going forward" |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 适用于维护一个被多模型共用的成熟 harness（Claude Code）并为新模型发布预调优的场景（"we spend time before each release optimizing the harness"）。 | "A system prompt change…" |
| 需要有按模型分别运行的、覆盖面足够广的评测集，才能发现单行 prompt 引起的 3% 级别回归。 | "A system prompt change…"；"Going forward" |
| 需要用户可复现的反馈渠道（/feedback、公开示例）作为发现问题的来源。 | "Going forward" 末段 |
| 所需算力/费用：未说明。 | — |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| 作者写到：内部使用与评测起初未复现；测试用的内部版本与公开版本不同；原评测集不够广；bug 处于 context 管理、API 与 extended thinking 的交叉处且只在角落情况出现。 | 开头段；"A caching optimization…"；"Going forward" |

| 核对发现："In combination with other prompt changes" 表明不止该行有变化，但其他改动是什么、各自影响多少未说明；回退后各模型的恢复数字未说明。 | "A system prompt change…" |
| 核对发现：上线前测试是否在所有受影响模型上分别进行未说明；effort 变更"slightly lower intelligence"也无数字。 | "A change to Claude Code's default reasoning effort" |

