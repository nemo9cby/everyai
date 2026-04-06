---
title: "Paper Digest: 2026-04-06"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的论文，不在 benchmark 花活，而在一个更根本的问题，post-training 到底该从哪里拿到足够密、但又不失真的学习信号。稀疏 outcome reward 太粗，纯 privileged distillation 又容易把目标带偏。今天最强的几篇 paper，基本都围着这条线打转，只是切入点不同，有的在讲 reasoning RL 的信号设计，有的在讲 code world model，有的在讲训练和 test-time compute 的联动。

## 今日 3 篇精选

### 1) Self-Distilled RLVR
- 链接: https://arxiv.org/abs/2604.03128
- 摘要速读: 这篇 paper 讨论的是 reasoning RL 里一个很现实的问题，大家都嫌 RLVR 的 reward 太 sparse，于是自然会想引入更 dense 的 teacher signal。问题在于，如果 teacher 拿着 privileged information，self-distillation 很容易发生 information leakage，长期训练还会越来越不稳。作者提出 RLSD，把 RLVR 保留为 update direction 的来源，也就是“模型该往哪学”依然由环境可验证反馈来决定，同时再用 self-distillation 提供 token-level policy difference，只负责调节 update magnitude，也就是“哪些 token 值得更大步地学”。
- 为什么重要: 这篇最有意思的地方，是它没有简单站队 sparse reward 或 dense distillation，而是把两者拆开处理。方向归 RLVR，力度归 self-distillation。这个拆法很像是在给 reasoning RL 做更细的 optimizer-interface 设计，而不是继续把所有学习信号混成一锅。

### 2) InCoder-32B-Thinking: Industrial Code World Model for Thinking
- 链接: https://arxiv.org/abs/2604.03144
- 摘要速读: 这篇 paper 瞄准的是工业代码场景，像芯片设计、GPU 优化、embedded systems，这些地方真正稀缺的不是代码本身，而是专家怎么围绕硬件约束、时序语义、执行反馈去推理。作者提出一个工业 code world model，用 Verilog simulation、GPU profiling 等 execution trace 训练模型去预测代码带来的行为后果，再配合 Error-driven Chain-of-Thought synthesis，生成和验证 reasoning traces。
- 为什么重要: 很多 code reasoning paper 其实还停留在 benchmark 题感里，像是在给 autocomplete 套一层思维链。这篇更像是在问，代码模型能不能先拥有一个“世界”，再去思考。只要这个方向成立，coding agent 的上限就不再只是会不会多想几步，而是能不能在真正的 execution semantics 里想对。

### 3) Test-Time Scaling Makes Overtraining Compute-Optimal
- 链接: https://arxiv.org/abs/2604.01411
- 摘要速读: 这篇 paper 试图把训练和推理放进同一个预算框架。作者提出 Train-to-Test scaling laws，在固定端到端 compute budget 下，联合优化模型大小、训练 token 数量和 inference 时的 sample 数。结论很直接，一旦把 repeated sampling 这种 test-time scaling 的成本认真算进去，compute-optimal 的预训练点会明显偏向更 overtrained 的区域，不再是传统 Chinchilla 风格给出的平衡点。
- 为什么重要: 这篇 paper 的价值在于提醒人们，现代模型的最优训练策略，已经不能只看 pretraining curve。部署时如果本来就要依赖 pass@k、search、sampling，那训练阶段的最佳配方也会跟着变。训练和推理，其实是同一个系统设计问题。

## 一句话结论
今天最强的研究信号可以概括成一句话，下一阶段真正拉开差距的，不只是更大的模型，而是谁更懂得把学习信号放在真正关键的位置。对 reasoning model 是这样，对 code model 也是这样，对训练和 test-time compute 的协同设计更是这样。