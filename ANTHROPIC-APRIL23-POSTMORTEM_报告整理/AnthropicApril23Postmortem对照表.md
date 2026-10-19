# ANTHROPIC-APRIL23-POSTMORTEM 对照表

报告：*An update on recent Claude Code quality reports*（Anthropic Engineering；2026-04-23；https://www.anthropic.com/engineering/april-23-postmortem）

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 事后排查：三项变更分别于 3 月 4 日、3 月 26 日、4 月 16 日上线，于 4 月 7 日回退、4 月 10 日修复、4 月 20 日回退；调查自 3 月初开始；4 月 20 日 v2.1.116 全部解决。其中 prompt 行是在 Opus 4.7 发布前"几周"为新模型调优 harness 时加入。 | 开头段 "three separate changes"；"A system prompt change to reduce verbosity" |
| 适用环境 | Claude Code、Claude Agent SDK、Claude Cowork 的生产流量；API 未受影响。涉及模型 Sonnet 4.6、Opus 4.6、Opus 4.7。 | 开头段 "The API was not impacted" |
| 轨迹来源 | 用户的真实会话（/feedback 反馈、线上可复现示例）；内部使用与内部评测；事后用 Opus 4.7 / 4.6 对引发 bug 的 PR 做 Code Review 回测。 | "Going forward"；"A caching optimization…" |
| 可见轨迹 | 用户报告的症状（"forgetful and repetitive"、UI 看似冻结、usage limit 消耗快）与评测分数；是否查看完整轨迹：未说明。 | 开头段三条；"A caching optimization…" |
| 交给优化器的输入 | 人工排查；对 system prompt 做逐行移除的 ablation（"removing lines from the system prompt"）并用更广评测集评估。 | "A system prompt change to reduce verbosity" |
| 诊断依据 | 内部评测；逐行 ablation；用户反馈；代码审查与各类自动测试（未能提前发现 bug）。 | "A system prompt change…"；"A caching optimization…" |
| 关注的失败 | (1) 思考过长致 UI 冻结、延迟与 token 用量不成比例；(2) 遗忘、重复、奇怪的工具选择，以及 cache miss 导致 usage limit 消耗快；(3) coding quality 下降。 | 开头段三条；各小节 |
| 禁止使用 | 未说明。 | — |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 三项 harness / 产品层变更（非模型）：默认 reasoning effort；context 管理中的 thinking 清除；system prompt 增加一行。模型与推理层未变。 | 开头段 "We never intentionally degrade our models" |
| prompt / 工具描述 | 4 月 16 日随 Opus 4.7 加入一行："Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail."；"In combination with other prompt changes" 损害 coding quality，4 月 20 日回退。 | "A system prompt change to reduce verbosity" |
| skill / 记忆 | 未说明。 | — |
| 控制逻辑（执行前） | 3 月 4 日默认 reasoning effort 由 high 改为 medium（减少长尾延迟、节省 usage limit）；4 月 7 日回退；现 Opus 4.7 默认 xhigh，其他模型默认 high。 | "A change to Claude Code's default reasoning effort" |
| 控制逻辑（执行后） | 3 月 26 日：会话空闲超过 1 小时后，用 clear_thinking_20251015 header + keep:1 清一次旧 thinking 以省 uncached token；bug 使其在该会话后续每一轮都清，只保留最近一个 reasoning block；4 月 10 日 v2.1.101 修复。 | "A caching optimization that dropped prior reasoning" |
| 明确不改 | API 与推理层未受影响；模型本身未变。 | 开头段 |
| 对照组 | system prompt 含某行 vs 不含该行（逐行 ablation）；Opus 4.7 vs Opus 4.6 做 Code Review 回测（4.7 找到 bug，4.6 没有）。 | "A system prompt change…"；"A caching optimization…" |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | token / 费用：未说明。时间：3 月初开始调查，起初难以与用户反馈的正常波动区分。 | 开头段 "challenging to distinguish from normal variation" |
| 诊断 | thinking 清除 bug 的根因花了"over a week"才发现并确认（角落情况 + 两项无关实验掩盖、难复现）。prompt 行的 ablation 成本：未说明。 | "A caching optimization…" |
| 搜索补丁 | 未说明（修复为回退/修 bug）。 | — |
| 验证 | prompt 行上线前经"multiple weeks of internal testing"且原评测集无回归；事后 ablation 用更广评测集。 | "A system prompt change to reduce verbosity" |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | 三项均于 4 月 20 日（v2.1.116）解决；修复后的量化指标：未说明。 | 开头段 |
| 对其他任务是否有效 | 未说明。 | — |
| 回归：流程层面 | 事前：原评测集无回归即上线。事后新增：对每次 system prompt 更改跑 per-model 的广评测集；持续逐行 ablation；prompt 更改审计工具；CLAUDE.md 要求模型特定更改门控到目标模型；soak period、更广评测、灰度发布；更多员工使用公开版本。 | "Going forward" |
| 回归：结果层面 | 更广评测集中一项评测显示 Opus 4.6 与 4.7 均下降 3%；评测名称、样本数、方差、3% 是绝对还是相对：未说明。 | "A system prompt change to reduce verbosity" |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 为 Opus 4.7 冗长问题写的 prompt 行，对 Opus 4.6 同样导致 3% 下降（4.6 与 4.7 各 3%）；effort 变更影响 Sonnet 4.6 与 Opus 4.6；thinking 清除 bug 影响 Sonnet 4.6 与 Opus 4.6；prompt 行影响 Sonnet 4.6、Opus 4.6、Opus 4.7。后续规定模型特定更改门控到目标模型。上线前测试是否只在单一模型上做：未说明。 | 开头段三条；"A system prompt change…"；"Going forward" |
| 换任务 | 未说明。 | — |
| 换执行环境 | 影响 Claude Code、Agent SDK、Cowork；API 不受影响；thinking bug 在多数 CLI 会话中被一项 thinking 显示方式的更改掩盖。 | 开头段；"A caching optimization…" |
| 换 Harness 实现 | 未说明。 | — |
| 补充 | 作者指出：三项变更各影响不同流量切片与时间表，叠加后看似"broad, inconsistent degradation"；内部使用与评测起初未复现；bug 通过了多轮人工/自动代码审查、单元测试、端到端测试、自动验证与 dogfooding。 | 开头段；"A caching optimization…" |
