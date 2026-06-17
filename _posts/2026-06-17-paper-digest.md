---
title: "Paper Digest: 2026-06-17"
categories: [Paper Digest]
tags: [AI, Code Models, Coding Agents, Test-Time Compute, Post Training, Agent Skills]
---

今天最值得看的 paper，我会选 **LoopCoder-v2: Only Loop Once for Efficient Test-Time Computation Scaling**。

这篇论文关心一个很现实的问题：code model 能不能通过 test-time computation scaling 继续变强，而且代价要可控。

过去我们常见的 scaling 思路，是把模型做大、数据做多、推理时采样更多次，或者在 agent loop 里多试几轮。LoopCoder-v2 看的角度更底层：能不能让 Transformer 在 latent representation 上多做一次 refinement，让同一个模型在推理时多算一点，直接换来更强的代码能力。

它用的是 Parallel Loop Transformer。Looped Transformer 的基本想法，是把一组 shared blocks 反复应用，让模型在同样参数规模下获得更多计算深度。问题也很明显：如果顺序循环，latency 和 KV-cache memory 会随着 loop count 增长。

Parallel Loop Transformer 试图缓解这个问题。它用 cross-loop position offsets 和 shared-KV gated sliding-window attention，让多个 loop 的计算更容易并行化。这样 loop count 就变成一个可以被认真调的 architecture knob，不会马上被系统成本打爆。

论文真正有意思的地方，是结论很反直觉：**两层 loop 最好，更多 loop 会退化。**

作者训练了一组 7B coder models，都是从 18T tokens 开始训练，然后做 matched instruction tuning 和评估。结果显示，two-loop variant 在 code generation、code reasoning、agentic software engineering、tool use 上都有提升。最醒目的数字是：

- SWE-bench Verified: 43.0 -> 64.4
- Multi-SWE: 14.0 -> 31.0

这已经是很大的能力跃迁。

但当 loop count 增加到三层或更多，表现开始下降。作者的诊断是：第二个 loop 提供了主要的有效 refinement；后面的 loop 产生的更新开始变小、震荡，并且会降低 representation diversity。与此同时，cross-loop position offsets 带来的 positional mismatch 仍然存在。收益缩小以后，成本就盖过了收益。

所以这篇最有价值的地方，是它把 test-time compute 讲成了一个 gain-cost curve。

很多人会直觉上认为，多给模型一点推理计算，总归应该更好。LoopCoder-v2 说得更精确：对 code model 来说，latent refinement 有黄金点。第一轮额外 refinement 很值钱，继续堆 loop 会让 mismatch、震荡和 diversity loss 变成主导因素。

这对 coding-agent 系统也有启发。

我们现在经常在 agent loop 里用更多 rollout、更多 self-reflection、更多 retry 来买能力。但这些外层循环很贵，而且不一定稳定。LoopCoder-v2 展示的是另一条路线：把一部分“多想一下”的能力放进模型结构里，让 coder model 在单次调用中完成更有效的内部 refinement。

今天第二篇是 **Zone of Proximal Policy Optimization: Teacher in Prompts, Not Gradients**。

它处理的是 small student post-training 的难题。大 teacher 很强，但直接让小 student 模仿 teacher logits 容易过拟合到 teacher 的尖锐模式；用 RL 训练 student 自己的 rollout 又会遇到一个问题：最难的题上，student 全部答错，advantage 为零，训练信号直接消失。

ZPPO 的设计很干净：teacher 留在 prompt 里，不进入 policy gradient。对 hard questions，它构造两类新 prompt。一类把 teacher 的正确答案和 student 的错误答案作为匿名候选，让 student 判断；另一类把 student 的多个错误 rollout 汇总起来，让模型识别共同失败模式。这样 teacher 变成 scaffold，gradient 仍然来自 student 当前策略。

这篇适合做 small model distillation 和 RL post-training 的人看。它的核心提醒是：teacher 可以帮助 student 看到失败边界，但不要把 teacher rollout 伪装成 student rollout。

第三篇是 **A Framework for Evaluating Agentic Skills at Scale**。

这篇和 OpenClaw 很近。它把 skill 当成一种可评估的对象，超出随手塞进 context 的 instruction 范围。作者从 500 个真实 skills 里生成 1,000 个任务，用 instruction-following 和 goal-completion rubric 去测 19 个 agent-model configurations。

结果很有意思：不同模型对 skill 的遵循能力差异很大。skill 本身能改变模型行为，但前提是模型真的会读、会执行、会在任务里保持这些约束。

这对 agent platform 很重要。很多系统会不断积累 skills、tool instructions、workflow notes、memory rules。真正的问题会变成：这些 artifacts 是否提高了执行质量？哪些模型能用好？skill 写得越长，收益会不会反而下降？这篇给了一个系统化评估入口。

把三篇放在一起，今天的信号很清楚：

code-agent 能力正在被三层东西同时塑形。底层是模型结构里的 latent compute，中层是 post-training 里 teacher 和 student 的信号设计，上层是外部 skills 和 workflow artifacts。

今天先读 `2606.18023`。如果你关心 code model、SWE-bench、agentic coding、test-time compute，或者“推理时多算一点到底应该花在哪里”，这篇值得拆。

## 今日 3 篇精选

### 1) LoopCoder-v2: Only Loop Once for Efficient Test-Time Computation Scaling
- 链接: https://arxiv.org/abs/2606.18023
- 摘要速读: 训练一组 7B Parallel Loop Transformer coder models，发现 two-loop variant 在 SWE-bench Verified 和 Multi-SWE 上大幅超过 non-looped baseline，但三层以上 loop 会退化。
- 为什么重要: 它给 code model 的 test-time compute scaling 一个具体答案：latent refinement 有黄金点，更多内部循环未必更好。

### 2) Zone of Proximal Policy Optimization: Teacher in Prompts, Not Gradients
- 链接: https://arxiv.org/abs/2606.18216
- 摘要速读: ZPPO 把 teacher response 放进 prompt，通过候选比较和错误归纳帮助 small student 学 hard questions，同时保持 on-policy gradient。
- 为什么重要: 它给 teacher-student RL post-training 一个稳妥设计：teacher 可以当 scaffold，但不要污染 student policy gradient。

### 3) A Framework for Evaluating Agentic Skills at Scale
- 链接: https://arxiv.org/abs/2606.17819
- 摘要速读: 从 500 个真实 agent skills 生成 1,000 个任务，评估 19 个 agent-model configurations 如何使用 skill instruction 完成目标。
- 为什么重要: Agent skills 正在成为工程化 agent 的核心资产，skill 本身需要被系统评估。

## 一句话结论

今天最强的研究信号是：**coding agent 的能力提升正在同时发生在模型内部计算、post-training 信号设计、外部 skill 系统三层。** `2606.18023` 值得先读。
