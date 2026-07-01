---
title: "Paper Digest: 2026-07-01"
categories: [Paper Digest]
tags: [AI, LLM Agents, Post-training, Discovery Agents, Coding Agents, RL]
---

今天最值得看的新 paper，我会选 **Evolution Fine-Tuning: Learning to Discover Across 371 Optimization Tasks**。

如果只按 HuggingFace 热度看，Dockerless 仍然是今天最贴近 coding agent 的强信号。但这篇昨天已经写过了，所以今天换一条同样重要的线：LLM 能不能把"反复改进一个解"学成自己的能力。

这篇 paper 的问题意识很直接。

过去一年，很多 LLM + search 系统已经在优化任务上做出很强结果，比如 GPU kernel design、数学猜想、scientific law discovery、combinatorial puzzles。常见做法是给模型套一个 evolutionary search scaffold：生成候选解，评估，挑好的，变异，再评估。

但这里有一个浪费：每次 search 结束后，trajectory 里的经验通常就被丢掉了。

模型也许找到了一个更快的 kernel，也许学会了某类数学构造该怎么局部修改，也许在一次失败里发现某种 backtrack 很有用。可下一道题来的时候，系统又从头开始。真正掌握"怎么进化一个解"的，还是外面的 scaffold。

Evolution Fine-Tuning 想把这部分经验写进模型。

作者提出 EFT，把 evolutionary search trajectories 转成监督数据，作为一个 mid-training phase。它训练模型学习这些行为：

- 哪一部分 solution 值得 mutate
- 什么时候该保守修改
- 什么时候该大幅探索
- 什么时候该 backtrack
- 如何把一个领域里的 search 经验迁移到新任务

他们构造了 Finch Collection：156K 条 trajectory，覆盖 10 个 domain、371 个 optimization tasks。模型规模从 2B 到 9B。

结果是，EFT 模型在 22 个 held-out tasks 上平均比 base model 高 **10.22%**。更有意思的是，当 EFT 模型再配合 test-time RL 时，它在两个 circle-packing task 上达到 state-of-the-art，并且在 Erdős minimum-overlap problem 上超过对应 base model。

我喜欢这篇 paper 的地方，是它把 agentic search 里很容易被忽略的一层拿了出来。

很多时候我们讨论 agent，会把重点放在工具、环境、planner、verifier 上。但如果模型本身完全不吸收 search 经验，每个新任务都要靠外部 scaffold 重新摸索，系统会很笨重。EFT 的野心是让模型先经历一个"练习阶段"，把通用的 discovery behavior 学进去。

这对 coding agents 也有启发。

写代码、修 bug、优化 kernel，本质上都是反复试探和修正的任务。真实流程里充满局部修改、回退、比较候选方案、保留有效差异、丢弃无用分支。EFT 提供了一个可以追的方向：把这些迭代行为整理成 training-time 数据，让它们能被模型反复学习。

今天第二篇是 **DOPD: Dual On-policy Distillation**。

这篇更偏 post-training 技术。

On-policy distillation 的基本想法是，让 student 在自己的 rollout 上接受 dense token-level supervision。这样可以减少 exposure bias，也能更细粒度地迁移 teacher 能力。

问题是，如果 teacher 或 student 带有 privileged information，很容易出现论文称为 **privilege illusion** 的失败模式：student 学到的表面行为，其实依赖了推理时拿不到的信息。

DOPD 的做法是按 token 动态决定监督来源和强度。它根据 advantage gap 和 relative probabilities，在 privileged teacher 与 privileged student policy 之间路由 token-level supervision。核心目标是把可迁移的能力传过去，同时避免把信息不对称当成能力差距来学。

这篇对做 post-training 的人很有用，因为它提醒了一件事：dense supervision 也需要清洗和校准。信号越密，越需要判断哪些 token 真的是能力，哪些只是隐藏信息泄露。

第三篇是 **SWE-INTERACT: Reimagining SWE Benchmarks as User-Driven Long-Horizon Coding Sessions**。

它重新设计了 coding-agent evaluation。

传统 SWE benchmark 通常一开始就给完整需求，然后看 agent 能不能 autonomous implementation。SWE-Interact 更接近真实开发：用户 simulator 一开始给模糊需求，之后逐步补充约束，检查 workspace，给反馈，要求修改，直到完整目标被交接出来。

结果很扎心：强模型在 single-turn SWE task 上能解大约 50%，但到了对应的 SWE-Interact task，只剩约 25%。

这说明单轮 coding benchmark 测不到很多真实能力，比如需求发现、记住上下文、根据反馈修正、避免过度行动、在多轮交互里保持设计一致。

把今天三篇放在一起，主题很清楚：

**agent training 要认真处理持续改进过程。**

EFT 关心模型如何从搜索轨迹中学习改进策略。DOPD 关心 post-training 里哪些 dense signal 真的可迁移。SWE-Interact 关心 coding agent 在多轮用户反馈下能不能稳住目标。

如果你关心 coding agents、SFT/RL、post-training、test-time search，今天先读 `2606.29082`。

## 今日 3 篇精选

### 1) Evolution Fine-Tuning: Learning to Discover Across 371 Optimization Tasks
- 链接: https://arxiv.org/abs/2606.29082
- 摘要速读: 把 evolutionary search trajectories 转成训练数据，让 LLM 学会如何迭代改进候选解，并在 371 个优化任务上构建 Finch Collection。
- 为什么重要: 它试图让模型吸收 discovery agent 的迭代改进经验，减少对一次性外部 search loop 的依赖。

### 2) DOPD: Dual On-policy Distillation
- 链接: https://arxiv.org/abs/2606.30626
- 摘要速读: 提出 advantage-aware dual distillation，在 privileged teacher 与 privileged student 之间动态路由 token-level supervision。
- 为什么重要: 它处理 on-policy distillation 中的 privilege illusion，适合关注 dense supervision 和 post-training 的团队。

### 3) SWE-INTERACT: Reimagining SWE Benchmarks as User-Driven Long-Horizon Coding Sessions
- 链接: https://arxiv.org/abs/2606.30573
- 摘要速读: 把 SWE benchmark 改造成多轮、用户驱动、需求逐步浮现的 coding session。
- 为什么重要: 它测的是 coding agent 在真实交互里的目标发现、需求保持和迭代修正能力。

## 一句话结论

今天最强的新信号是，**下一阶段 agent training 要把持续改进一个解的过程本身当作训练对象。** `2606.29082` 值得先读。
