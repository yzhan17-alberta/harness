# AnthropicLongRunningHarness 文章总结

> 报告：*Harness design for long-running application development*（Prithvi Rajasekaran，Anthropic Labs；2026-03-24；https://www.anthropic.com/engineering/harness-design-long-running-apps）

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：如何让 Claude 产出高质量前端设计，并在无人干预下完成多小时的完整应用构建；早期 prompt 工程与 harness 设计"eventually hit ceilings"。 | 开头段 |
| 子问题 1：长任务中模型随 context 填满失去连贯、出现 context anxiety；自我评估时过度乐观。 | "Why naive implementations fall short" |
| 子问题 2：harness 跑通后如何在不降低性能的前提下简化；哪些组件在新模型上仍 load-bearing。 | "Iterating on the harness" |
| 形式化目标：无；作者提出的原则是"every component in a harness encodes an assumption about what the model can't do"，这些假设会随模型进步而过时，值得压力测试。 | "Iterating on the harness" |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| 被优化对象：三 agent harness（planner / generator / evaluator）的 prompt、评分准则与编排（sprint、contract、context reset、evaluator 时机）。 | "The architecture" |
| 方法本身：受 GAN 启发的 generator–evaluator 循环；全栈版加 planner；简化时逐组件移除并人工检视。 | 开头段；"Iterating on the harness" |
| 任务领域：前端设计（HTML/CSS/JS 页面）；全栈应用（2D 复古游戏制作器；浏览器 DAW）。 | "Frontend design…"；"Running the harness"；"Results…" |
| 底层模型：早期 harness Sonnet 4.5；V1 Opus 4.5（当时最佳编码模型）；V2 运行模型未明说（推断：Opus 4.6）。 | "The architecture"；"Running the harness"；"Removing the sprint construct" |
| 优化器/判官/专家模型：evaluator 为独立 agent，配 Playwright MCP（是否与 generator 同一模型：未明说，只由 V1 "I used Claude Opus 4.5" 整体表述暗示）；最终裁判为作者人工。 | "Frontend design…"；"The architecture"；"Running the harness" |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 全栈 V1 | 一个 prompt："Create a 2D retro game maker with features including a level editor, sprite editor, entity behaviors, and a playable test mode."；solo 与 full harness 各一次运行；planner 扩成 16 个功能、十个 sprint；Sprint 3 合同 27 条准则。 | "Running the harness" |
| 全栈 V2 | 一个 prompt："Build a fully featured DAW in the browser using the Web Audio API."；一次运行，三轮 build/QA。 | "Results from the updated harness" |
| 前端设计 | 多个用户 prompt（文中举例 Dutch art museum 网站）；每次 5–15 轮迭代；运行数未说明。 | "Frontend design…" |
| 评测数据集 / 切分 / 去污染 | 无数据集；无切分；去污染不适用，未说明。 | — |

## 4. 核心变量

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索/被操纵的对象 | harness 组件：context reset、sprint 构造、sprint contract、evaluator 时机（每 sprint vs 末尾一次）、planner、评分准则措辞与权重、in-app agent prompting。 | "The architecture"；"Removing the sprint construct" |
| 对比条件 | Solo 单 agent vs Full harness（Opus 4.5，同一 prompt）；前端：有准则 prompting vs 无 prompting baseline。 | "Running the harness"；"Frontend design…" |
| 消融条件 | 第一次：大幅砍掉多组件 + 新想法（失败）；之后逐组件移除：去 sprint（保留）、去 planner（恢复）、evaluator 保留。 | "Iterating on the harness"；"Removing the sprint construct" |
| 结果指标 | 无量化结果指标；人工试用应用、阅读日志；evaluator 分数（有硬阈值，但未作为结果报告）；时长与费用。 | 全文 |
| 控制条件 | 未说明（V1 与 V2 任务不同，模型不同）。 | — |
| 过程统计量 | 时长与费用（V1：20 min/$9 vs 6 hr/$200；V2 总计 3 hr 50 min/$124.70，分项见成本表）；前端 5–15 轮、最长 4 小时；builder 无 sprint 连续运行超过 2 小时；evaluator 两轮 QA 的具体反馈文本。 | "Running the harness"；"Results from the updated harness" |

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| 搜索/分析循环：planner 扩 spec → generator 构建（V1 按 sprint，先协商 contract）→ evaluator 用 Playwright 测试并按准则打分 → 不达阈值则反馈给 generator 重做；V2 去 sprint，evaluator 末尾单次 pass，DAW 实际跑了三轮 build/QA。 | "The architecture"；"Results from the updated harness" |
| 关键设计：生成与评估分离（调一个怀疑型 evaluator 比让 generator 自我批评更可行）；四条可评分准则、重设计与原创性；few-shot 校准 evaluator；sprint contract 桥接高层 spec 与可测实现；文件通信；Opus 4.5 起去掉 context reset 改用 SDK 自动 compaction。 | "Why naive implementations fall short"；"Frontend design…"；"The architecture" |
| 接受/选择规则：evaluator 任一准则低于硬阈值即 sprint 失败；简化时以作者人工判断"what impact it had on the final result"决定去留。 | "The architecture"；"Iterating on the harness" |
| 规模：每种配置文中各一次运行；两个全栈任务；前端多个 prompt。 | 全文 |
| 评估设计：作者手动打开并试用应用、阅读日志；与 solo 运行对比。 | "Running the harness" |
| 统计显著性检验 | 未说明。 |
| 重复次数 / 随机种子数 | 未说明（文中每种配置只描述一次运行）。 |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| 前端：准则 + 独立 evaluator 循环使评分随迭代提升后趋平；第一轮输出已明显好于无 prompting 的 baseline；作者有时更偏好中间轮次而非最后一轮；复杂度随轮次上升。 | "Frontend design…" |
| 准则措辞直接塑造输出风格（"museum quality"导致视觉趋同）；Dutch art museum 例子第 10 轮出现 3D 房间式的创造性跳跃。 | "Frontend design…" |
| V1（Opus 4.5）：Full harness 6 hr / $200 vs Solo 20 min / $9（>20x），但 solo 版核心游戏无法玩，harness 版可玩且更精致；planner 扩成 16 功能、十个 sprint；Sprint 3 合同 27 条准则。 | "Running the harness" |
| 开箱即用的 Claude 是差的 QA agent（发现问题后自我说服通过、测试浅）；经数轮调 prompt 后可用，但仍漏掉小布局问题与深层 bug。 | "Running the harness" |
| 一次性大幅简化失败：无法复现原性能且无法判断哪些组件 load-bearing；改为逐组件移除。 | "Iterating on the harness" |
| Opus 4.6 发布后去掉 sprint；planner 保留（否则 generator under-scope）；evaluator 保留，改为末尾单次 pass；evaluator 是否值得取决于任务是否超出模型可靠独立完成的边界。 | "Removing the sprint construct" |
| V2（DAW）：3 hr 50 min / $124.70，builder 无 sprint 连续运行超过 2 小时；QA 两轮仍发现真实缺口（clip 不能拖动、无乐器面板、录音为 stub 等）；最终应用具备 arrangement view、mixer、transport，agent 可端到端作一段短曲。 | "Results from the updated harness" |
| 结论性建议：用目标模型做实验、读 trace；复杂任务可分解并用专门 agent；新模型发布时重审 harness，去掉不再 load-bearing 的部分并加新能力；harness 组合空间随模型进步"moves"而非缩小。 | "What comes next" |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 任务复杂到超出模型可靠独立完成的边界时 harness/evaluator 才值得其成本；边界内的任务 evaluator 是"unnecessary overhead"。 | "Removing the sprint construct" |
| 需要把主观质量转成可评分准则，并用 few-shot 校准 evaluator；evaluator 需要能实际操作产物（Playwright MCP）。 | "Frontend design…"；"The architecture" |
| 需要人工阅读 trace 与 evaluator 日志做调参与组件取舍。 | "Running the harness"；"Iterating on the harness" |
| 基于 Claude Agent SDK（自动 compaction、编排）。 | "Frontend design…"；"The architecture" |
| 所需算力/费用：V1 全 harness 约 $200 / 6 小时；V2 约 $124.70 / 3 hr 50 min；前端运行最长 4 小时；按 token 计费。 | 成本表 |

## 8. 研究局限

| 局限 | 原文依据 |
|---|---|
| 作者自认：evaluator 仍有 verification headroom（漏小布局问题与深层 bug）；Claude 不能听，QA 对音乐无效；DAW 远非专业软件，agent 作曲能力弱；游戏版仍有产品直觉缺口（流程不引导先建 sprite）。 | "Running the harness"；"Results from the updated harness" |
| 统计显著性/方差：无任何量化结果指标，每种配置一次运行，无重复、无方差、无检验。 | 全文 |
| 内部核对：成本表跨版本比较的是不同任务（游戏制作器 vs DAW）和不同模型，V1→V2 的时长/费用下降混合了模型、harness 与任务三个因素。 | 两张成本表 |
| 内部核对：evaluator 被描述为移到"a single pass at the end of the run"，但 DAW 成本表显示三轮 QA（R1/R2/R3）；两者关系未解释。 | "Removing the sprint construct"；"Results from the updated harness" 成本表 |
| 内部核对：context reset 是因相信 Opus 4.5 不再需要而去掉，并未展示保留它会有害；evaluator 的保留基于"continued to add obvious value"，未描述移除实验。 | "The architecture"；"Removing the sprint construct" |
| 未说明：DAW 运行使用的模型（只能从上下文推断为 Opus 4.6）；逐组件移除各跑了几次；evaluator 分数的具体数值；"old harness on new model unchanged"的量化表现。 | — |
