# RETHINKING-HARNESS-EVAL 对照表

论文：*Rethinking the Evaluation of Harness Evolution for Agents*（Yike Wang, Huaisheng Zhu, Zhengyu Hu, Yige Yuan, Zhengyu Chen, Shakti Senthil, Hannaneh Hajishirzi, Yulia Tsvetkov, Noah A. Smith, Pradeep Dasigi, Teng Xiao；Allen Institute for AI / University of Washington / Independent；arXiv:2607.12227，2026 年 7 月）

注：这是一篇评测方法论文章。它不提出新的 harness 优化方法，而是把已有的 harness evolution（以 AHE 实例化）和自己定义的 harness scaling，放到与 test-time scaling 基线（Parallel Sampling、Sequential Refinement）匹配预算的协议下重新测量。下表中"方法"一词指论文实际运行的 Harness Evolution / Harness Scaling 两种 harness 修改流程。

## 1. 场景与输入

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 适配时机 | 四种方法都在推理/评测期介入，模型固定，各自消耗同一预算 K：Parallel Sampling 花在 K 条独立轨迹；Sequential Refinement 花在 K 轮轨迹修订；Harness Evolution 花在跨任务共享的 harness 更新（离线搜索一批任务后得到 ĥ，再对目标任务采样）；Harness Scaling 花在单任务实例级的 harness 更新（运行中边跑边改）。EdgeBench 上 Harness Scaling 是周期性触发：每 3 次提交或 120 分钟触发一次 meta agent 更新。 | §3；§3.4；§3.5；§5.3；App. C.1 |
| 适用环境 | Terminal-Bench 2.1（89 个终端任务；相对 2.0 修复了 28 个任务）；ARC-AGI-3 全部 25 个公开游戏（64×64 网格、16 色、逐关卡计分）；EdgeBench 的 Interactive Games & Simulators 类 5 个任务（dcss、nethack、openrct2、openttd、wesnoth，agent 写 Lua / Squirrel / Python / ES5 JS bot 由模拟器评分）。 | §4.1；§5.1；App. B.1；App. C.1 |
| 轨迹来源 | 被评测的 policy 自己在当前 harness 下产生：Terminal-Bench 上 Claude Opus 4.6 / GPT-5.4 / GPT-5.4 mini；ARC-AGI-3 上 Claude Opus 4.8 / GPT-5.6 Terra；EdgeBench 上 Claude Opus 4.8 / GPT-5.4。摘要器 Φ、meta agent、self judge 都由与 policy 相同的底层模型实现。§4.4 用训练集轨迹演化、验证集选 harness、测试集报分。 | §3.1；§3.2；§3.5；§4.1；§4.4；§5.1；Table 6 |
| 可见轨迹 | Harness Evolution：经验库 C_k 累积每轮的 (harness, 该 harness 下全部 rollout)，有单测时还含每条 rollout 的 0/1 结果；meta agent 读到的是 Φ(C_{k−1})。Harness Scaling：meta agent读到 Φ(上一轮 harness, 上一条轨迹[, 结果])。Φ 由 AHE 的 Agent Debugger 实现：每条轨迹消息存为文件，按任务分组，输出带 pass/fail 标注的逐任务分析报告和基准级总览；原始轨迹保留可按需查看。EdgeBench 上由 summarizer 压缩轨迹，meta agent 需要时可读原始轨迹。 | §3.1；§3.4；§3.5；App. A.1 "Summarization Map"；App. C.1 "Implementation" |
| 交给优化器的输入 | Harness Evolution：h_k = M(Φ(C_{k−1}))，k=2…K，起点 h_1。Harness Scaling：h_k = M(x, Φ(h_{k−1}, y_{k−1})) 或含 R(y_{k−1}, g)。ARC-AGI-3 上采用 VISTA 的"自由更新"：harness 更新以工具形式（read_skill、write_skill、write_tool、run_tool、write_hooks、read/write_guide、read/write_working）暴露给 agent，随时可调。 | §3.4；§3.5；§5.2；App. B.1 |
| 诊断依据 | 无单测设定：只靠 agent 自己的判断（Parallel Sampling 用 self judge J 选最优候选；Harness Evolution 直接取最后一个 harness h_K）。有单测设定：R(y, g) ∈ {0,1} 既作反馈也作最终轨迹选择的 oracle。Agent Debugger 负责找每个失败的根因 / 成功的模式。ARC-AGI-3 用官方 RHAE 分数；EdgeBench 用隐藏 judge 的分数与反馈。 | §3.2–§3.5；§4.2；§4.3；App. A.1；App. B.1；App. C.1 |
| 关注的失败 | App. A.2 列出 meta agent 针对的反复失败类：交付过晚/预算意识不足、改动前未复制易碎状态、结束前未复核任务约束、超时、过度轮询、环境未配好；Fig. 2 还列出易碎的安装/构建、不安全的清理、依赖版本未固定、数据格式假设错误、破坏性检查、embedding prompt 类型配错、试图削弱测试。作者同时指出存在"稳定的困难失败核心"（深度领域推理或 harness 之外的约束），不受 harness 修改影响。基础设施异常（沙盒崩溃、API 超时）记 r=0 计入失败。 | App. A.2；Fig. 2；App. A.1 "Evaluation Metrics" |
| 禁止使用 | AHE 的 explore agent 被关闭：不允许从外部检索已为该基准调好的 harness。§4.4 要求搜索任务与评测任务不相交（45 train / 10 val / 34 test）。"无单测"设定下不提供 R。其他明令禁止项（如 task id、ground truth）：未说明。 | §3.4；App. A.1 "Agentic Harness Engineering (AHE)"；§4.4 |

## 2. 修改对象

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 总体 | 改的是模型外的 harness。Harness Evolution 产出一个跨任务共享的 harness（AHE 可演化 prompt、tools、middleware、skills、long-term memory）；Harness Scaling 为单个任务实例改 harness。ARC-AGI-3 上 [VISTA] 只改记忆，[full harness] 还改 skills、tools、hooks。EdgeBench 上"记忆仍是 harness 被修改的主要界面"。Parallel Sampling / Sequential Refinement 不改 harness，改的是轨迹。 | §2；§3；§3.4；§3.5；§5.3；App. B.1 |
| prompt / 工具描述 | Harness Evolution 的首选动作通常在 prompt 层：向系统 prompt 和长期记忆加行为规则；也改工具层：纠正误导性的工具说明、在命令执行时注入恢复提示。Harness Scaling 把具体失败的知识（已知 bug、实现模板、验证检查、文件路径、命令序列）写进下一轮 prompt 或长期记忆，并用紧凑命令计划替代重复轮询/探索命令。App. A.2 还提到会"纠正"工具指引、"替换"低效命令序列，并考察"哪些改动被保留或回滚"，因此编辑不全是新增，但删除/回滚未量化。 | App. A.2；Fig. 2 |
| skill / 记忆 | AHE 可改 skills 与 long-term memory。ARC-AGI-3：记忆为 GUIDE.md（跨关卡持久的游戏模型）与 WORKING.md（每关重置的草稿），[full harness] 另可写 skill（如 visual_substitution.md、loop_puzzle.md、block-and-delivery）。Table 12：每局写 skill 4.3（Opus 4.8）/ 3.8（GPT-5.6 Terra），读 skill 2.0 / 12.6；开放全 harness 后记忆操作从 24.2→20.0、35.4→27.0 每局（−17% / −24%）。 | §2；App. B.1；App. B.2；Table 12 |
| 控制逻辑（执行前） | 原文未按"执行前/后"划分。可归入执行前的有：工具层在命令执行时注入恢复提示、prompt 中规定命令顺序与显式消息预算、允许更长的 shell 超时、跳过不必要的环境探测（推断：按 App. A.2 描述归类）。 | App. A.2；Fig. 2 |
| 控制逻辑（执行后） | middleware 层做运行时强制：轮次预算跟踪器（到阈值提醒交付）、截断过大的工具输出、finalization gate（交付物缺失或未验证时阻止结束）；Harness Scaling 还加提交前的语法检查、单测、模仿验证器的本地检查。ARC-AGI-3 [full harness] 可写 hooks（如 after_play 事件的 blocked_move 规则），但每局几乎不写（0.2 / 0.0）。 | App. A.2；App. B.2；App. B.3；Table 12 |
| 明确不改 | 模型权重不改（test-time scaling 定义；"given a fixed agent policy"）。Parallel Sampling 与 Sequential Refinement 全程使用固定 harness h；Sequential Refinement [Claude Code / Codex] 与 [VISTA*] 的 harness 在整局中保持不变。基准任务与验证器不改。 | §2；§3；§3.2；§3.3；App. B.1 |
| 对照组 | Terminal-Bench：direct sampling using initial harness（do-nothing）、Parallel Sampling、Sequential Refinement，初始 harness 与 AHE 相同的最小 harness（仅一个 bash 工具，无 skills / middleware / 记忆）。ARC-AGI-3：Sequential Refinement [Claude Code CLI / Codex CLI]（仅供参考）和 [VISTA*]（VISTA 关闭记忆演化）。EdgeBench：Sequential Refinement 用 Claude Code CLI / Codex CLI 作固定 harness。 | §4.1；App. A.1 "Initial Harness"；§5.2；§5.3；App. B.1 |

## 3. 成本

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 收集数据 | 预算以 rollout 数 K 计：主实验 K=5，消融 K=10（仅 Claude Opus 4.6）；AHE 每 harness 每任务 m=1 条 rollout 以对齐预算；所有结果取 2 次独立运行均值。每条 rollout 在独立的 E2B 远程沙盒中执行；code agent 最大 300 轮、单轮最大生成 128k tokens、上下文 200k。ARC-AGI-3：每局上限 2,000 个动作。EdgeBench：12 小时 wall-clock。token 总量与费用：未说明。 | §4.1；App. A.1 "Configuration"；Table 6；App. A.3；App. B.1 "Cost"；App. C.1 "Cost"；Table 14 |
| 诊断 | Agent Debugger 与 policy 同模型，最大 25 轮、上下文压缩阈值 0.75、重试 3 次；其调用次数 / token / 费用：未说明。 | Table 6 |
| 搜索补丁 | Meta Agent 与 policy 同模型，最大 500 轮。Terminal-Bench 上**不计**修改 harness 的成本，理由是相对生成新轨迹很便宜且计入只会强化结论。EdgeBench 上轨迹摘要与 harness 编辑时间计入 12 小时预算，更新期间 agent 空闲。 | Table 6；App. A.1 "Cost"；App. C.1 "Cost" |
| 验证 | Terminal-Bench：有单测设定下每条 rollout 跑单测；§4.4 用 10 个验证任务选 harness。EdgeBench：agent 可随时提交给隐藏 judge（冷却 120 s），主机侧每 1,800 s 另做一次评测（结果不展示给 agent）；judge CPU 4 核 / 内存 8 GB。ARC-AGI-3：按官方 RHAE 计分。验证单独的费用：未说明。 | §4.3；§4.4；App. C.1；Table 14 |

## 4. 成功标准

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 原失败是否修好 | Terminal-Bench 无单测（pass@1 均值）：direct 68.2、Parallel Sampling 72.3、Sequential Refinement 69.3、Harness Evolution 67.4、Harness Scaling 71.8；GPT-5.4 上 Harness Evolution 75.3→69.7。有单测（pass@1 / pass@5 均值）：direct 67.6 / –、Parallel 82.0 / 82.0、Sequential 76.9 / 87.1、Harness Evolution 69.7 / 79.0、Harness Scaling 75.5 / 85.6。作者判断：harness 修改的收益只在可多选时（pass@5）出现，pass@1 几乎不动；"稳定的困难失败核心"未被修好。ARC-AGI-3（RHAE 均值）：Opus 4.8 [VISTA*] 43.3→[VISTA] 59.3→[full harness] 77.5（+79.0% 相对）；GPT-5.6 Terra 21.1→25.9→38.2（+81.0%）。EdgeBench（最佳分均值）：Opus 4.8 48.3→54.7（+13.3%）；GPT-5.4 45.9→49.8（+8.5%）。 | Table 1；Table 2；§4.3；App. A.2；Table 4；§5.2；Table 5；§5.3 |
| 对其他任务是否有效 | 同基准 held-out（45 train / 10 val / 34 test，训练集带单测，按验证集选 harness，测试集报 pass@1）：Claude Opus 4.6 63.3→64.5（+1.2）、GPT-5.4 72.1→72.1（+0.0）、GPT-5.4 mini 54.4→53.0（−1.4）、均值 63.2→63.2（+0.0）。K=10 时 Opus 4.6 63.3→63.0（−0.3）。跨基准迁移：未测。 | §4.4；Table 3；Table 9 |
| 回归：流程层面 | Terminal-Bench 上没有非回归门控：无单测时直接取最后一个 harness h_K；有单测时取经验库中平均结果 R̄(h_k) 最高的 harness。App. A.2 提到考察"哪些改动被保留或回滚"，但未给出规则或数量。EdgeBench：meta agent 每次更新可选择不改；改动需通过 validation 后写 sync_request 才安装；报告的是 best-so-far，作者称反馈"也防止回归"。ARC-AGI-3：未说明。 | §3.4；App. A.2；§5.1；App. C.1 "Implementation" |
| 回归：结果层面 | 报告了退步：Table 1 中 Harness Evolution 使 GPT-5.4 从 75.3 降到 69.7，Sequential Refinement 使 GPT-5.4 从 75.3 降到 73.0；Table 3 中 GPT-5.4 mini −1.4；Table 5 中 GPT-5.4 在 nethack 50.4→49.0、openttd 51.6→50.5，Opus 4.8 在 wesnoth 88.0→86.0 退步；Table 4 中逐游戏也有退步（如 sk48 Opus 4.8 [VISTA] 23.3→[full harness] 14.0）。重复次数为 2 次独立运行取均值。方差数值、置信区间、显著性检验：未说明（Table 8 标题称差距"在 run-to-run 方差之内"，但未给方差）。 | Table 1；Table 3；Table 4；Table 5；§4.1；Table 8 |

## 5. 迁移范围

| 子项 | 原文内容 | 原文依据 |
|---|---|---|
| 换模型 | 未测。每个模型各自演化自己的 harness（code agent、Agent Debugger、meta agent 均为同一模型），没有"用 A 模型演化的 harness 给 B 模型用"的实验。作者在 §5 提到结果"可能只是反映当前模型还不够强以修订整个 harness"。 | Table 6；§5 |
| 换任务 | 仅测同基准内 held-out：Table 3 / Table 9，平均增益 +0.0 / −0.3，作者结论是"编码了任务特定捷径而非更好的 harness 设计原则"。ARC-AGI-3 与 EdgeBench 按设计是逐任务优化，"不是寻找单一可迁移的 harness"。 | §4.4；Table 3；Table 9；§5.1 |
| 换执行环境 | 未测。三个基准各用不同后端（E2B 沙盒 / Docker / Kubernetes），但没有把演化出的 harness 换到另一环境运行的实验。 | App. A.1；Table 10；Table 14 |
| 换 Harness 实现 | 未测。起点 harness 不同（Terminal-Bench 最小 bash harness；ARC-AGI-3 VISTA*；EdgeBench Claude Code CLI / Codex CLI），但没有把演化结果移植到另一 harness 实现的实验。 | App. A.1；§5.2；§5.3 |
| 补充 | 作者写到：Terminal-Bench 结果可能源于模型不够强、训练期已针对 Terminal-Bench 优化、分数已高、或 Terminal-Bench 对 harness 不敏感（最小 shell + 基础 prompt 已够）；Terminal-Bench 上的结论不外推到 ARC-AGI-3 / EdgeBench（那里 Harness Scaling 优于 Sequential Refinement）；Table 12 的计数"衡量使用而非价值"；EdgeBench 上 meta agent 偏重单路径深挖、有时忽略广度；Terminal-Bench 上不计 harness 修改成本。 | §5；§5.2；§5.3；App. B.2；App. A.1 "Cost" |
