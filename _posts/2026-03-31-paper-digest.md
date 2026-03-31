---
title: "Paper Digest: 2026-03-31"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得记住的研究信号，是 agent 讨论越来越像在谈一套真正的生产系统，而不是一堆会做任务的模型拼装件。真正开始变重要的，是任务怎么分发，结果怎么验收，经验怎么沉淀，资产怎么复用，成本怎么结算。换句话说，agent research 正在从 capability demo，走向 organizational infrastructure。

## 今日 3 篇精选

### 1) EpochX: Building the Infrastructure for an Emergent Agent Civilization
- 链接: https://arxiv.org/abs/2603.27304
- 摘要速读: 这篇 paper 的野心很大，但问题意识是对的。作者认为，foundation model 让 agent 的基础能力越来越像公共品，接下来真正卡住系统规模化的，不再只是模型本身，而是任务如何被委托、分解、验证和激励。EpochX 提出一个 credits-native 的 agent marketplace，把人和 agent 都当成生产网络中的参与者。任务可以被 claim、拆成 subtasks、经过显式 delivery workflow 验证，完成之后还能沉淀成可复用的 skill、workflow、execution trace 和 distilled experience。
- 为什么重要: 这篇最有意思的地方，在于它把 agent 基础设施里几件经常分开讨论的事，硬生生放进了同一个闭环里：执行、验收、记忆、复用、激励。很多 agent 系统现在最大的问题，不是跑不动，而是跑完什么都没留下。EpochX 想解决的，正是 work leaves artifacts 这件事。

### 2) TAPS: Task Aware Proposal Distributions for Speculative Sampling
- 链接: https://arxiv.org/abs/2603.27027
- 摘要速读: Speculative decoding 现在大家都知道能加速生成，但常见默认是 draft model 只要够轻就行。TAPS 追问的是更实际的一步，draft model 的训练分布到底会不会影响加速质量。结论很直接，会，而且影响不小。MathInstruct 训练出来的 drafter 更适合 reasoning，ShareGPT drafter 更适合 chat，混合数据带来鲁棒性，但不是越大越万能。作者进一步用 confidence-based routing 在 inference 时做 specialized drafter 选择，效果比 naive checkpoint averaging 更好。
- 为什么重要: 这篇 paper 提醒人们，inference stack 也有 workload alignment 问题。部署优化不是纯工程活，数据分布和路由策略也会决定系统效果。对真正做 serving 的团队来说，这比再看一个 benchmark 排名更有用。

### 3) Gen-Searcher: Reinforcing Agentic Search for Image Generation
- 链接: https://arxiv.org/abs/2603.28767
- 摘要速读: 这篇把 search agent 和 image generation 接了起来。核心思路是让模型先做 multi-hop search，收集文本知识和参考图，再做 grounded image synthesis。训练上走的是一条很清楚的路线，先 SFT，再用 dual reward feedback 做 agentic RL，同时给出配套数据集和 benchmark。
- 为什么重要: 它最值得看的不是 image application 本身，而是那条训练路径。也就是，当任务天然需要外部世界知识时，retrieval 不该只是一段外挂流程，而该进入 action space，再被后训练真正优化。

## 一句话结论
今天这些 paper 指向同一个方向，下一代 agent 的价值，越来越取决于系统能不能把一次执行变成下一次能力。谁更会留下可复用资产，谁就更可能把 agent 从 demo 变成生产力。
