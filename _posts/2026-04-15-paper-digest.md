---
title: "Paper Digest: 2026-04-15"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的 paper，是一篇把 post-training 里一个常被使用、却少有人真正讲透的方法拆开来看的工作。**Rethinking On-Policy Distillation of Large Language Models: Phenomenology, Mechanism, and Recipe** 讨论的是 OPD，on-policy distillation。这个方向这半年越来越像基础设施，大家都在用，但很多时候用法更像经验偏方，teacher 换大一点、prompt 调一调、数据铺一铺，跑出来有效就继续，跑不动就归因给实现细节。這篇 paper 的价值，在于它试图回答一个更根本的问题，为什么有些 OPD 很灵，有些几乎学不到东西。

## 今日 3 篇精选

### 1) Rethinking On-Policy Distillation of Large Language Models: Phenomenology, Mechanism, and Recipe
- 链接: https://arxiv.org/abs/2604.13016
- 摘要速读: 这篇 paper 系统研究 OPD 的训练动力学，提出两个决定成败的条件。第一，student 和 teacher 需要有相容的 thinking pattern。第二，teacher 必须真的带来 student 训练中没有见过的新能力，而不只是分数更高但分布相近。作者用 weak-to-strong reverse distillation 证明，同一家族的 1.5B 和 7B teacher 在 student 看来甚至可能几乎分布不可区分。进一步从 token level 分析，成功的 OPD 往往表现为在 student 已访问状态上，对高概率 token 的逐步对齐，而且真正起决定作用的是一个很小但承载了 97% 到 99% 概率质量的 shared token set。论文还给了两个补救失败 OPD 的实用策略，off-policy cold start 和 teacher-aligned prompt selection。
- 为什么重要: 这篇 paper 最有价值的地方，是它把一个被广泛使用的 recipe 变成了可解释的机制。很多人默认更强的 teacher 自然能教出更强的 student，但作者指出，若 teacher 没有把 student 推到一个真正新的能力区域，distillation 很可能几乎学不到东西。这个判断很重要，因为它直接影响 teacher 选择、数据设计、甚至是否值得做 OPD。

### 2) KnowRL: Boosting LLM Reasoning via Reinforcement Learning with Minimal-Sufficient Knowledge Guidance
- 链接: https://arxiv.org/abs/2604.12627
- 摘要速读: KnowRL 面向 RLVR 的 reward sparsity 问题，不是继续给模型塞更多 hint，而是把 hint 设计成一个 minimal-sufficient guidance 问题。它把提示拆成 atomic knowledge points，再用 constrained subset search 选出紧凑但交互感知的子集做训练。作者还发现一个很有意思的 pruning interaction paradox，删掉一个 knowledge point 可能有益，但一起删多个反而可能伤害表现。最终模型在 1.5B 规模上超过多个强基线。
- 为什么重要: 这篇 paper 的贡献不只是“hint 也能提升 RL”，而是它提醒人，指导信息不是越多越好。真正关键的是哪部分 guidance 对 learning trajectory 最有效，哪些只是噪声和冗余。

### 3) ClawGUI: A Unified Framework for Training, Evaluating, and Deploying GUI Agents
- 链接: https://arxiv.org/abs/2604.11784
- 摘要速读: ClawGUI 给 GUI agent 提供了一个从训练、评测到部署的统一框架，覆盖并行虚拟环境、真实设备、标准化 benchmark 评估，以及多平台落地。作者训练出的 2B agent 在 MobileWorld GUI-Only 上超过同规模基线。
- 为什么重要: 它的意义更偏基础设施。GUI agent 这条线并不只缺模型，更多时候缺的是稳定环境、统一评测和能真正落地的闭环。

## 一句话结论
今天最强的研究信号是，post-training 已经越来越不像“把 teacher 换大一点，把 recipe 叠厚一点”这种粗暴工程。真正起作用的是学习结构本身，teacher 和 student 的分布关系是否合适，hint 是否真是最小充分，agent 训练环境是否稳定可复现。OPD 这篇 paper 最值得看，因为它让一个已经被大量使用的方法，第一次显得没那么像玄学。