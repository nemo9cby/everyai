---
title: "Paper Digest: 2026-05-15"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, Post-Training]
---

今天最值得看的 paper，我会给 **Self-Distilled Agentic Reinforcement Learning**。

这篇 paper 打中的问题很实在。大家都知道 RL 已经成了 agent post-training 的主轴，但一旦任务变成多轮交互，reward 往往太粗，反馈来得太晚，错误还会一路滚雪球。很多时候，agent 不是完全不会做，而是学不到足够细的修正信号。

SDAR 的思路，我觉得比很多“再加一个复杂模块”的方案更值得注意。它没有试图推翻 RL 主体，而是保留 RL 做主干，再把 self-distillation 变成一个**受控的辅助目标**。

具体来说，作者借用了带 privileged context 的 teacher branch，去提供更密的 token-level guidance。但关键不在“有 teacher”，而在**不要把 teacher 信号当圣旨**。multi-turn agent 里，teacher 的 negative rejection 也可能只是因为 skill retrieval 不稳、工具使用不顺、或者上下文条件不完整。如果硬搬过去，训练很容易被带偏。

所以 SDAR 加了一个 gate。它把 detached token-level signal 映射成 sigmoid 权重，对 teacher 明显认可的正向 token 强化蒸馏，对那些可能带噪声的负向 rejection 则只做软抑制。这个处理很重要，因为它承认了 agent 训练里的一个现实，**dense supervision 当然好，但 dense noise 也会更快把系统带歪。**

paper 里的结果也不差。在 ALFWorld、WebShop、Search-QA 上，它相对 GRPO 都有明显提升，而且还能避开 naive GRPO+OPSD 那种不稳定混训问题。这个点让我更愿意把它当成“可迁移的 recipe”，而不只是一次 benchmark win。

我觉得这篇 paper 真正值得看的地方，是它代表了一种更成熟的 agent post-training 心态。

大家过去很容易把 agent 能力问题理解成两类：要么 base model 不够强，要么 reward 设计不够好。但多轮 agent 其实还有第三类问题，**训练信号在执行链条里会持续失真。** 只靠最终 reward 太粗，只靠 privileged teacher 又太容易过拟合脏信号。SDAR 的价值，就在于它开始认真处理这个中间层。

对做 tool-use agent、search agent、GRPO/agent RL 的团队来说，这篇很值得读。它逼着人重新想几个问题：

- token-level auxiliary signal 到底该怎么和 trajectory-level RL 共存？
- multi-turn skill retrieval 带来的噪声，应该怎么进入训练视角？
- teacher signal 什么时候该放大，什么时候该降权？
- agent post-training 的下一步，是否会越来越像“信号工程”而不只是“reward engineering”？

如果这个方向继续走下去，agent 训练的真正分水岭，可能不是谁先把 reward 做出来，而是谁更会管理那些夹在 rollout、tool use、teacher judgment、policy update 之间的中间信号。这正是 SDAR 今天最值得优先看的原因。

## 今日 3 篇精选

### 1) Self-Distilled Agentic Reinforcement Learning
- 链接: https://arxiv.org/abs/2605.15155
- 摘要速读: 用 gated self-distillation 给 multi-turn agent RL 补上更细的 token-level guidance，同时避免把 teacher 噪声生硬灌进策略更新。
- 为什么重要: 它直面 agent RL 里最棘手的部分，长期交互中的稀疏奖励和失真监督。

### 2) Achieving Gold-Medal-Level Olympiad Reasoning via Simple and Unified Scaling
- 链接: https://arxiv.org/abs/2605.13301
- 摘要速读: 用 reverse-perplexity curriculum SFT、两阶段 RL 和 test-time scaling，做出一个金牌级 olympiad reasoning recipe。
- 为什么重要: 真正有价值的是这套 reasoning post-training 配方，而不只是 medal headline。

### 3) WildClawBench: A Benchmark for Real-World, Long-Horizon Agent Evaluation
- 链接: https://arxiv.org/abs/2605.10912
- 摘要速读: 在真实 CLI harness 里做长时程 agent 评测，任务更长、工具更真、验收更看 side effects，最强模型也只有 62.2%。
- 为什么重要: 它把“agent 看起来很强”和“agent 在真实运行时真的靠谱”之间的差距重新拉回台面。

## 一句话结论
今天最强的研究信号是，**agent 的难点越来越集中在训练信号和真实运行时之间的那层摩擦。** 从 SDAR 的 gated dense supervision，到 WildClawBench 的原生运行时评测，再到 SU-01 这种系统化 reasoning recipe，焦点都在变得更具体、更工程化。`2605.15155` 值得优先读，因为它最贴近现实 agent post-training 的痛点。