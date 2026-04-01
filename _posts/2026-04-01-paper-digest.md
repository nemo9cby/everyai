---
title: "Paper Digest: 2026-04-01"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得记住的研究信号，不是某个模型又多刷了几个点，而是 post-training 终于开始更认真地处理 credit assignment 这件事。长 reasoning trace 里，真正值得学的 token 并不均匀，coding task 里的思考也不会只发生在开头。换句话说，推理能力不只是“能不能想”，更是“什么时候该想，哪里该被奖励”。

## 今日 3 篇精选

### 1) FIPO: Eliciting Deep Reasoning with Future-KL Influenced Policy Optimization
- 链接: https://arxiv.org/abs/2603.19835
- 摘要速读: FIPO 盯住的是 ORM-style reasoning RL 里一个很核心、也很容易被糊弄过去的问题，credit assignment 太粗。很多方法最后给出一个 outcome reward，然后把这个全局 advantage 差不多均匀摊到整条轨迹上。问题在于，长链条 reasoning 里真正关键的逻辑转折点，和那些只是把句子往前推的 token，并不该拿到同样的学习信号。FIPO 把 discounted future-KL 引入 policy update，让 token 的权重和它对后续 trajectory 的影响挂钩。论文里在 Qwen2.5-32B 上，把平均 reasoning length 从约 4,000 tokens 拉到 10,000+，AIME 2024 Pass@1 也明显提升。
- 为什么重要: 这篇最有价值的地方，在于它把“长 CoT 为什么会卡住”说得更具体了。很多时候不是模型不会继续想，而是训练信号根本没有告诉它，哪些位置真的值得更用力。对后训练来说，这比再讨论一次“要不要 RL”更实。

### 2) Think Anywhere in Code Generation
- 链接: https://arxiv.org/abs/2603.29957
- 摘要速读: 这篇 paper 提出一个很贴近 coding task 直觉的观点，代码生成里的思考，不该只发生在 output 之前。真正写代码时，复杂度往往是边写边暴露出来的，所以模型需要一种可以在任意 token 位置临时插入 reasoning 的机制。作者先做 cold-start imitation，教模型模仿这种 reasoning pattern，再用 outcome-based RL 让模型自己学会什么时候、哪里值得停下来想一想。结果在 LeetCode、LiveCodeBench、HumanEval、MBPP 上都很强。
- 为什么重要: 它提醒人们，reasoning policy 可能是 task-specific 的。对 code 来说，upfront thinking 很可能天然不够，因为真正难的部分经常出现在 implementation 的中段。这对 code model 和 coding agent 都是个很实在的启发。

### 3) daVinci-LLM: Towards the Science of Pretraining
- 链接: https://arxiv.org/abs/2603.27164
- 摘要速读: 这是少见的、愿意把 pretraining 过程认真摊开的工作。作者训练了一个 3B 模型，8T tokens，做了 200+ ablations，系统讨论 data processing depth、不同 domain 的 saturation dynamics、composition balance 和 curriculum 这些问题，还把完整处理流程和训练过程尽量公开。
- 为什么重要: 这篇 paper 的价值，不在于喊一个更大的数字，而在于把 pretraining 拉回一种可累积的科学研究状态。尤其是它强调 data processing depth 可能和 data volume 一样重要，这对所有过度迷恋 post-training 神迹的人，都是一个必要提醒。

## 一句话结论
今天最强的几篇论文都在说同一件事，模型能力的上限，越来越取决于训练信号分配得够不够细、够不够贴任务本身。谁更懂得把学习信号打在真正关键的位置，谁更可能把“会推理”变成“推理得更深、更准、也更会用在代码上”。
