---
title: "Paper Digest: 2026-06-24"
categories: [Paper Digest]
tags: [AI, Agents, World Models, Qwen, RL, Post Training]
---

今天最值得看的 paper，我会选 **Qwen-AgentWorld: Language World Models for General Agents**。

这篇论文把 agent training 里一个关键对象摆到了台前：环境。

如果一个 agent 要在网页、代码仓库、GUI、工具系统里规划和执行，它面对的是一连串 state transition：

- 当前观察是什么
- 下一步 action 是什么
- 环境会怎样变化
- 哪些反馈代表成功
- 哪些错误会把任务带进死路

真实环境当然最可信，但它慢、贵、难复现，也很难大规模并行。纯文本合成环境又容易太浅，训练出来的 agent 很可能只学到表面格式。

Qwen-AgentWorld 的路线是训练一个 **language world model**，让模型用语言形式模拟 agentic environment 的变化。

论文发布了两个模型：**Qwen-AgentWorld-35B-A3B** 和 **Qwen-AgentWorld-397B-A17B**。它们覆盖 7 个 agentic domains，并且用 long chain-of-thought reasoning 来预测下一步环境状态。

训练 pipeline 分三段：

1. **CPT**：用 state transition dynamics 和 augmented professional corpora 注入通用 world modeling 能力。
2. **SFT**：激活 next-state-prediction reasoning，让模型学会沿着 action 和 observation 推演。
3. **RL**：用 hybrid rubric-and-rule rewards 提高模拟 fidelity。

数据规模也很硬。作者用了超过 **10M environment interaction trajectories**。评估上，他们构建了 **AgentWorldBench**，来自 5 个 frontier models 在 9 个已有 benchmarks 上的真实交互。

这篇 paper 对我来说最有价值的地方，是它把 agent RL 的环境瓶颈转成了一个可以训练、可以评估、可以复用的模型问题。

Qwen-AgentWorld 有两个用法。

第一，它可以作为 decoupled environment simulator。你不用每次都去真实环境里交互，可以先用 world model 批量模拟成千上万个环境，给 agentic RL 提供更高吞吐的训练场。

第二，它可以作为 unified agent foundation model 的 warm-up。也就是说，world-model training 本身可以让 agent 在进入下游任务前先学会环境动态、反馈模式和长程推演。

这很像给 agent 先建立一种环境直觉。

今天很多 agent 论文还在讨论更复杂的 orchestration、更多工具、更长 prompt。Qwen-AgentWorld 关心的是更底层的问题：如果 agent 没有对环境动态的可迁移理解，它的规划和 RL 都会被真实交互成本卡住。

这对 post-training 很重要。

做 coding agent、terminal agent、GUI agent、browser agent，本质上都绕不开一个问题：训练信号从哪里来，环境反馈怎么规模化，失败轨迹怎么变成可学习的数据。world model 提供了一个新的中间层。它既不需要完全依赖真实环境，也不把训练退回静态 instruction tuning。

今天第二篇是 **NatureBench: Can Coding Agents Match the Published SOTA of Nature-Family Papers?**。

NatureBench 选了 90 个来自 Nature-family papers 的真实 scientific tasks，目标是测试 coding agents 能不能在科学问题上接近或超过论文里的 published SOTA。

它的关键工程是 **NatureGym**。NatureGym 会从 source paper 构造标准化、containerized 的 per-task environment，尽量解决研究型 benchmark 常见的环境碎片化问题。

结果挺冷静。10 个 frontier agent configurations 在禁用 web search 的协议下，最强模型只有 **17.8%** 的任务超过 published SOTA（g>0.1 criterion）。

更有意思的是成功路径。agent 成功时，主要靠的是把科学任务翻译成熟悉的 supervised prediction problem；失败时，主要问题是 wrong method choice 和 compute budget 不够。

这篇适合给所有 research agent 的乐观叙事降温。能复现环境、能跑代码、能做 method translation，已经很有价值，但这和真正的 scientific invention 还有距离。

第三篇是 **OpenThoughts-Agent: Data Recipes for Agentic Models**。

它关心 agentic model training data 怎么做。已有 open efforts 往往只盯一个 benchmark，比如 SWE-Smith、SERA、Nemotron-Terminal。OpenThoughts-Agent 做的是一个更系统的 data curation pipeline。

作者跑了超过 **100 个 controlled ablations**，然后组了一个 **100K examples** 的训练集，fine-tune Qwen3-32B。结果在 7 个 agentic benchmarks 上达到 **44.8%** 平均准确率，比 Nemotron-Terminal-32B 高 **3.9 points**。

这篇的实践启发很直接：agentic data recipe 不能只靠一个任务生成器。task source、diversity、filtering、mixture、scaling 都应该被当成可测的 knobs。

把三篇放在一起，今天的信号很集中：

agent 能力的下一步很可能取决于训练基础设施。更好的 simulator，更真实的 executable benchmark，更开放的数据 pipeline，都会直接影响 agent 在真实环境里的上限。

今天先读 `2606.24597`。

如果你关心 agent RL、post-training、world models、code agents，或者想理解 Qwen 这类团队怎么把 agent environment 变成训练资产，这篇值得拆。

## 今日 3 篇精选

### 1) Qwen-AgentWorld: Language World Models for General Agents
- 链接: https://arxiv.org/abs/2606.24597
- 摘要速读: 训练语言 world models 来模拟 agentic environments，覆盖 7 个 domains，使用 CPT、SFT、RL 三阶段 pipeline，并基于 10M+ environment interaction trajectories 构建。
- 为什么重要: 它把 agent RL 的环境交互瓶颈改造成一个可训练的 simulator 问题，可以用于 scalable simulation 和 agent warm-up training。

### 2) NatureBench: Can Coding Agents Match the Published SOTA of Nature-Family Papers?
- 链接: https://arxiv.org/abs/2606.24530
- 摘要速读: 从 Nature-family papers 抽取 90 个 scientific coding tasks，并用 NatureGym 构造 containerized environments 来评估 agent 是否能接近 published SOTA。
- 为什么重要: 最强 agent 只在 17.8% 的任务上超过 SOTA，说明 research coding agent 主要还停留在 method translation 层面。

### 3) OpenThoughts-Agent: Data Recipes for Agentic Models
- 链接: https://arxiv.org/abs/2606.24855
- 摘要速读: 一个开放的 agentic model data curation pipeline，包含 100+ ablations、100K examples 和 Qwen3-32B fine-tuning。
- 为什么重要: 它给 agent post-training 提供了更系统的数据工程 recipe，在 7 个 benchmarks 上超过现有 strongest open-data agentic model。

## 一句话结论

今天最强的研究信号是，**agent training 的核心基础设施正在变得更清晰：simulator、benchmark、data recipe，三者都会决定 agent 的真实上限。** `2606.24597` 值得先读。
