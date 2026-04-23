---
title: "Paper Digest: 2026-04-23"
categories: [Paper Digest]
tags: [AI, RL, Post-Training, Agents, LLM]
---

今天最值得看的 paper，是一篇把后训练里一个经常被默认、却很少被讲清楚的问题直接拎出来的工作。**Near-Future Policy Optimization** 关心的核心，不是再堆一个更复杂的 RL 框架，也不是去找一个更强的外部 teacher，而是一个非常实际的问题，**当 RLVR 或 GRPO 开始进入平台期，模型到底该向谁学。**

这篇 paper 最妙的地方，在于它把这个问题回答得既直观，又足够工程化。现有 mixed-policy 方法常常卡在两头都不舒服。你如果引入外部 teacher trajectory，质量通常很高，但分布离当前 policy 太远，学起来方差大、吸收效率差。你如果只 replay 过去训练过程中自己的旧轨迹，分布当然更近，可这些样本往往已经不够“新”，能提供的有效学习信号也被天花板卡住了。

作者给出的答案很漂亮，**让模型向“未来一点点的自己”学习。** 也就是从同一次训练 run 里，挑一个稍后一点的 checkpoint，把它产生的 trajectory 拿回来喂给当前 policy。这个想法听起来甚至有点朴素，但越想越对。因为 near-future self 天然满足两个很难同时满足的条件，它确实比当前 policy 更强，可又没有远到像外部 teacher 那样让分布错位得厉害。换句话说，它更像一个真正“够得着”的老师。

这篇 paper 真正让我觉得有分量的地方，还不只是 intuition，而是它把这种 intuition 变成了一个很清楚的训练原则。作者用 trajectory quality 和 variance cost 的平衡来解释为什么 near-future checkpoint 特别合适，然后通过 NPO 和 AutoNPO 展示了这件事在实际训练里能成立。尤其 AutoNPO 很值得看，它不是固定死规则，而是根据在线训练信号去判断什么时候该介入、该选哪个 future checkpoint。这个设计很有味道，因为它让方法从一个静态 trick，变成一个可以嵌进真实后训练流水线的 adaptive mechanism。

对后训练来说，这件事的重要性不小。很多时候我们以为 RL 平台期意味着奖励设计不够、采样不够、模型不够大，或者干脆觉得该换更强 teacher 了。但这篇 paper 提醒人，问题也可能出在**学习信号的来源不对**。不是所有更强的轨迹都 equally useful，真正高效的那些轨迹，往往既要比你强一点，又不能强得离谱。NPO 抓住的就是这个张力。

如果这个思路能继续被验证，它会给很多 RL-for-LLM 系统一个很实际的改法。你不一定需要额外蒸馏一个昂贵 teacher，也不一定要不停囤积越来越长的 replay buffer。你可以先问一个更本地、更节制的问题，当前 policy 的下一个好老师，会不会就在它自己训练轨迹的前方不远处。

## 今日 3 篇精选

### 1) Near-Future Policy Optimization
- 链接: https://arxiv.org/abs/2604.20733
- 摘要速读: 这篇 paper 提出 NPO 和 AutoNPO，让当前 policy 去学习同一次训练 run 里稍后一点 checkpoint 生成的 trajectory。重点在于这种 near-future self 同时更强、又更近，能更好平衡学习价值和分布偏移带来的方差成本。
- 为什么重要: 它抓住了 RLVR 训练里一个真正痛的地方，轨迹到底该从哪里来。这个答案很实用，而且比“找更强外部 teacher”更容易嵌进现有训练系统。

### 2) DR-Venus: Towards Frontier Edge-Scale Deep Research Agents with Only 10K Open Data
- 链接: https://arxiv.org/abs/2604.19859
- 摘要速读: DR-Venus 做了一件很有吸引力的事，用大约 10K open-data，把一个 4B deep research agent 磨到很强。方法上结合 agentic SFT、agentic RL、长轨迹数据重采样，以及基于 information gain 的 turn-level reward。
- 为什么重要: 这篇 paper 的价值，在于它说明小模型 agent 不是只能做演示。只要训练配方够认真，4B 规模也能把长程任务执行力拉到一个很值得正眼看的水平。

### 3) A Self-Evolving Framework for Efficient Terminal Agents via Observational Context Compression
- 链接: https://arxiv.org/abs/2604.19572
- 摘要速读: TACO 研究 terminal agent 的 context 膨胀问题，把 observation compression 做成一个会自我演化的模块，不再依赖固定 prompt 或手工规则去总结终端反馈。
- 为什么重要: terminal agent 真跑起来以后，最先拖后腿的常常不是“不会做”，而是上下文越来越臃肿、越来越贵。TACO 碰到的是一个很真实的系统瓶颈。

## 一句话结论
今天最强的研究信号是，后训练里最值钱的改进，越来越来自**更好的学习信号设计**，不是更花哨的口号。`2604.20733` 值得重点看，因为它给了一个很扎实的提醒，模型最合适的老师，很多时候未必在外面，可能就在它未来一点点的自己那里。