# ANTHROPIC-LONG-RUNNING-HARNESS 对照表

报告：*Harness design for long-running application development*（Prithvi Rajasekaran，Anthropic Labs；2026-03-24；https://www.anthropic.com/engineering/harness-design-long-running-apps）

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 离线、人工：先用 Opus 4.5 跑完整 harness（V1），结果"bulky, slow, and expensive"后着手简化；期间 Opus 4.6 发布，进一步促使减少 scaffolding；作者建议每当新模型发布就重审 harness。 | "Iterating on the harness"；"What comes next" |
| 适用环境 | 多小时自主全栈开发：一句话 prompt → 完整应用（2D 复古游戏制作器；浏览器 DAW）；前面还有前端设计实验（HTML/CSS/JS 页面）。技术栈 React、Vite、FastAPI、SQLite（后改 PostgreSQL）。 | "Running the harness"；"Results from the updated harness"；"The architecture" |
| 轨迹来源 | 作者自己运行 harness 与 solo 单 agent 产生；每种配置文中各描述一次运行。V1 用 Opus 4.5；早期 harness 用 Sonnet 4.5；V2（DAW）运行时的模型未明说（推断：Opus 4.6，依据"Removing the sprint construct"围绕 4.6 展开且 DAW 段称"without the sprint decomposition that Opus 4.5 had needed"）。 | "Running the harness"；"Results from the updated harness" |
| 可见轨迹 | 作者阅读日志/trace（"Reading through the logs"）、evaluator 的反馈文件、最终应用的人工试用；evaluator 本身通过 Playwright MCP 操作页面并截图。 | "Running the harness"；"Frontend design…" |
| 交给优化器的输入 | 人工：evaluator 调参时读 evaluator 日志、找与作者判断分歧处并改 QA prompt；简化时以 Opus 4.6 发布博客的能力声明为先验决定可去掉什么。 | "Running the harness"；"Iterating on the harness" |
| 诊断依据 | 作者人工检视每次移除组件后的最终结果；evaluator 对各准则打分，任一准则低于硬阈值则 sprint 失败。 | "Iterating on the harness"；"The architecture" |
| 关注的失败 | 长任务失去连贯、context anxiety 提前收尾；自我评估偏乐观（"confidently praising the work"）；evaluator 把发现的问题说服自己不重要、测试浅；generator under-scope、stub 功能。 | "Why naive implementations fall short"；"Running the harness"；"Removing the sprint construct" |
| 禁止使用 | 未说明。 | — |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 整个多 agent harness（planner / generator / evaluator 的 prompt 与编排）；模型本身不改。 | "The architecture" |
| prompt / 工具描述 | 四条评分准则（Design quality、Originality、Craft、Functionality）写进 generator 与 evaluator 的 prompt，前两条权重更高；evaluator 用带分数拆解的 few-shot 校准；全栈版准则改为 product depth、functionality、visual design、code quality；V2 新增让 generator 构建能通过工具驱动应用的 in-app agent 的 prompting。措辞会左右输出（"museum quality"导致视觉趋同）。 | "Frontend design…"；"The architecture"；"Removing the sprint construct" |
| skill / 记忆 | planner 可读 frontend design skill 生成视觉设计语言；agent 间用文件通信；早期 harness 用结构化 handoff artifact 跨 session 传递状态。 | "Running the harness"；"The architecture"；"Why naive implementations fall short" |
| 控制逻辑（执行前） | planner 把 1–4 句 prompt 扩成完整 spec（要求 ambitious、聚焦产品而非细节实现）；V1 中 generator 与 evaluator 在每个 sprint 前协商 sprint contract；V2 删除 sprint 构造。 | "The architecture"；"Removing the sprint construct" |
| 控制逻辑（执行后） | evaluator 用 Playwright MCP 像用户一样点击测试 UI、API、数据库状态，按阈值判定；V1 每 sprint 评一次，V2 改为运行末尾单次 pass；context reset（Sonnet 4.5 需要）在 Opus 4.5 上整体去掉，改用 Agent SDK 自动 compaction；generator 每 sprint 末自评；git 版本控制。 | "The architecture"；"Removing the sprint construct" |
| 明确不改 | 未说明（全文只描述 harness 改动，未提模型训练）。 | — |
| 对照组 | Solo 单 agent（同一 prompt）vs Full harness（Opus 4.5）；逐组件移除后与原结果比较（人工）；前端：无 prompting 的 baseline。 | "Running the harness"；"Iterating on the harness"；"Frontend design…" |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | V1（Opus 4.5，游戏制作器）：Solo 20 min / $9；Full harness 6 hr / $200（"over 20x more expensive"）。V2（DAW）：Planner 4.7 min / $0.46；Build R1 2 hr 7 min / $71.08；QA R1 8.8 min / $3.24；Build R2 1 hr 2 min / $36.89；QA R2 6.8 min / $3.09；Build R3 10.9 min / $5.88；QA R3 9.6 min / $4.06；总计 3 hr 50 min / $124.70。前端：每次生成 5–15 轮迭代，完整运行最长 4 小时。 | "Running the harness" 成本表；"Results from the updated harness" 成本表；"Frontend design…" |
| 诊断 | 未说明（逐组件移除实验各跑了几次、花费多少均未给出）。 | — |
| 搜索补丁 | evaluator 调参"several rounds"；in-app agent prompting"took real iteration"；均未量化。 | "Running the harness"；"Removing the sprint construct" |
| 验证 | 未说明。 | — |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | 无量化指标；人工判断：solo 版游戏"nothing responded to input"，harness 版可以移动实体并游玩（物理有瑕疵、有过不去的墙）；DAW 版具备 arrangement view、mixer、transport 并可由 agent 作曲，但"far from a professional"。evaluator 分数存在但未作为结果数字报告。 | "Running the harness"；"Results from the updated harness" |
| 对其他任务是否有效 | 未说明（V1 与 V2 分别只在游戏制作器与 DAW 上运行，未交叉）。 | — |
| 回归：流程层面 | 第一次简化：大幅砍掉并尝试新想法，无法复现原性能且无法判断哪些部分 load-bearing；随后改为"removing one component at a time"并检视结果。移除 planner 导致 generator under-scope，故保留；evaluator 因"continued to add obvious value"保留，未描述移除 evaluator 的实验。 | "Iterating on the harness"；"Removing the sprint construct" |
| 回归：结果层面 | 无数字；每种配置一次运行；无方差、无检验。 | 全文 |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 同厂商连续升级，harness 人工再适配：Sonnet 4.5 → Opus 4.5 去掉 context reset（"Opus 4.5 largely removed that behavior"）；Opus 4.5 → 4.6 去掉 sprint、evaluator 改为末尾单次（"Opus 4.6… good reason to expect… less scaffolding"）。无冻结 harness 跨模型的数字。 | "The architecture"；"Iterating on the harness"；"Removing the sprint construct" |
| 换任务 | V1 跑游戏制作器，V2 跑 DAW；前端设计实验另有 Dutch art museum 等 prompt；未在同一任务上比较两版 harness。 | "Running the harness"；"Results from the updated harness" |
| 换执行环境 | 未说明。 | — |
| 换 Harness 实现 | 未说明（均基于 Claude Agent SDK）。 | — |
| 补充 | 作者写到：evaluator 仍漏掉小布局问题与深层嵌套 bug，"more verification headroom"；Claude 不能听，QA 对音乐品味无效；evaluator 是否值得取决于任务相对模型能力边界的位置，不是固定的是/否；随着模型进步 harness 组合空间"moves"而非缩小。 | "Running the harness"；"Removing the sprint construct"；"Results…"；"What comes next" |
