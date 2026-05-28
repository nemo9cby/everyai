---
title: "Paper Digest: 2026-05-28"
categories: [Paper Digest]
tags: [AI, Agents, Post-Training, Reinforcement Learning, Tool Use]
---

今天最值得看的 paper，我会给 **Agent Explorative Policy Optimization for Multimodal Agentic Reasoning**。

这篇来自 NVIDIA，核心问题很清楚：agentic reasoning 里有两种动作，一种是在模型内部继续思考，另一种是调用外部工具。后者更接近真实 agent 能力，但训练起来也更不稳定。

作者把这个问题叫 **Thinking-Acting Gap**。

在标准 GRPO 训练里，他们观察到两个现象。第一，模型只在大约 30% 的 rollouts 里尝试 tool use。第二，在尝试 tool use 的样本里，约 40% 的问题会出现整组 tool-using rollouts 全错。这样一来，真正需要学习的 tool call 分支反而拿不到有效训练信号。

AXPO 的做法很直接：当某个 tool-using subgroup 全错时，保留前面的 thinking prefix，重新采样 tool call 和后续 continuation。再配合 uncertainty-based prefix selection，把探索预算集中到最值得修正的位置。

这个设计重要的地方在于，它没有把 tool use 当成普通 token 继续训。tool use 的失败结构不同：可能前面的 reasoning 已经足够好，问题出在什么时候调用工具、调用什么工具、如何利用工具返回结果。AXPO 把这个分叉点单独拿出来优化。

实验结果也有意思。在九个 multimodal benchmarks 和三个 Qwen3-VL-Thinking 尺度上，SFT+AXPO 相比 SFT+GRPO 在 8B 上平均提升约 +1.8pp Pass@1 和 +1.8pp Pass@4。更关键的是，8B 的 SFT+AXPO 在 Pass@4 上超过了 32B Base。

这对 coding agent 和 computer-use agent 都很有参考价值。

很多 agent failure 的根因在 action branch 探索不足。比如代码 agent 知道该查文件，但不会稳定地选对文件；browser agent 知道要操作 UI，但在关键一步失手；tool-use 模型知道外部工具有用，但探索成本高、失败密度大。AXPO 给出的启发是：post-training 需要围绕这些分叉点设计采样和奖励，不能只把整条 trajectory 当成同质样本。

今天另外两篇也值得一起看。

**LearnWeak** 关注小型 computer-use agents 的领域专精。它用强参考 agent 诊断学生模型在目标 domain 里的弱点，然后自动合成针对性任务和监督数据。它还把 planning error 和 execution error 拆开训练。OSWorld 上，LearnWeak 给 EvoCUA-8B 和 OpenCUA-7B 带来约 +11 个点的平均提升。这个方向很务实：不要盲目造合成数据，先问学生模型到底弱在哪里。

**Laguna M.1/XS.2 Technical Report** 是另一篇 agentic coding foundation model 技术报告。M.1 是 225.8B total / 23.4B active 的 MoE，XS.2 是 33.4B total / 3B active。论文把模型放在 Model Factory 里讲，从 versioned data、training、evaluation、inference、post-training 到 quantization 都覆盖，并在 SWE-bench Verified、SWE-bench Multilingual、SWE-Bench Pro 和 Terminal-Bench 2.0 上评估。XS.2 权重已经 Apache 2.0 发布。

把今天三篇放在一起看，信号很清楚：agent post-training 正在变得更诊断化。AXPO 诊断 tool-use exploration collapse，LearnWeak 诊断学生 agent 的领域弱点，Laguna 把 coding-agent model factory 作为完整生产系统来描述。

所以今天先读 `2605.28774`。如果你关心 SFT/RL、tool-use agents、coding agents，或者想把 GRPO 用在更真实的 action trajectory 上，这篇值得认真看。

## 今日 3 篇精选

### 1) Agent Explorative Policy Optimization for Multimodal Agentic Reasoning
- 链接: https://arxiv.org/abs/2605.28774
- 摘要速读: 提出 AXPO，通过保留 thinking prefix 并重新采样失败的 tool-use 分支，改善 GRPO 在 agentic reasoning 中的探索不足。
- 为什么重要: 它把 tool use 的训练断点讲清楚了，适合迁移到 coding agent、browser agent 和多工具 agent 的 post-training。

### 2) Learn from Weaknesses: Automated Domain Specialization for Small Computer-Use Agents
- 链接: https://arxiv.org/abs/2605.28775
- 摘要速读: 用强参考 agent 发现小型 computer-use agent 的领域弱点，再自动生成针对性任务和监督数据。
- 为什么重要: 它把 weakness diagnosis 变成训练数据引擎，比盲目合成 trajectory 更贴近实际部署。

### 3) Laguna M.1/XS.2 Technical Report
- 链接: https://arxiv.org/abs/2605.27605
- 摘要速读: 发布面向 long-horizon agentic coding 的 MoE 模型族，并介绍覆盖数据、训练、评测、推理和 post-training 的 Model Factory。
- 为什么重要: 它提供了另一个 foundation model team 如何工程化 coding-agent 训练栈的参考样本。

## 一句话结论
今天最强的研究信号是：**agent post-training 要围绕失败分叉点做诊断和采样。** `2605.28774` 值得先读。
