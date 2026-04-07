---
title: "Paper Digest: 2026-04-07"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的论文，落点非常明确，coding model 真正缺的，也许不是再多一点“会说”的 reasoning，而是更像程序员那样，先在脑子里跑一遍代码，再决定自己写得对不对。今天几篇强 paper 放在一起看，会发现一个共同方向，大家都在试图把模型从只会产出答案，往会模拟过程、会修正错误、会更新信念那边推。

## 今日 3 篇精选

### 1) Self-Execution Simulation Improves Coding Models
- 链接: https://arxiv.org/abs/2604.03253
- 摘要速读: 这篇 paper 直接瞄准 coding model 一个很核心的短板，模型会写代码，但并不真正擅长预估自己写出来的代码会怎么执行。作者的办法很干脆，先用 grounded in true execution 的自然语言 execution traces 做 supervised fine-tuning，让模型学会 step-by-step 模拟程序运行；再加上带 verifiable rewards 的 reinforcement learning，让这种 execution simulation 真正服务于 competitive programming 求解。最终模型可以在 test-time 对多个候选解做 self-verification，还能根据自己模拟出来的执行结果迭代 self-fix。
- 为什么重要: 这篇最打动人的地方，是它没有把代码推理继续做成“更长的 reasoning trace”，而是把 attention 拉回 execution semantics 本身。代码任务的 truth，很多时候不在文字里，而在运行结果里。模型如果能先形成一个 internal executor，再去写和改代码，coding agent 的上限会明显变高。

### 2) Unifying Group-Relative and Self-Distillation Policy Optimization via Sample Routing
- 链接: https://arxiv.org/abs/2604.02288
- 摘要速读: 这篇 paper 讨论 reasoning RL 里一个非常现实的问题，GRPO 这种 reward-driven 方法稳定，但 credit 太粗；SDPO 这种 self-distillation 信号更密，但训练久了又容易崩。作者提出 SRPO，不再争谁替代谁，而是按 sample 分流，正确样本交给 GRPO 继续做 reward-aligned reinforcement，错误样本交给 SDPO 做 targeted token-level correction，再用 entropy-aware weighting 压掉不可靠的 distillation target。
- 为什么重要: 真正有价值的地方，不是它又造了一个 acronym，而是它开始把 post-training signal 当成可路由、可调度的资源来看。模型已经答对的时候，需要的信号和答错的时候，本来就不该一样。这个视角很像更成熟的 post-training stack 会走的方向。

### 3) ClawArena: Benchmarking AI Agents in Evolving Information Environments
- 链接: https://arxiv.org/abs/2604.04202
- 摘要速读: 这篇 paper 想解决的是 persistent assistants 的现实难题，信息源会冲突，历史结论会被新消息推翻，用户偏好往往也不是直接说出来，而是从纠错里慢慢显露。ClawArena 用多通道 session、workspace files、动态更新和 hidden ground truth 设计 benchmark，不只考 reasoning，还考 belief revision、implicit personalization 和 workspace grounding。
- 为什么重要: 现在很多 agent benchmark 还停留在静态世界里，一次问答、一个答案、一个标准真相。现实中的 agent 根本不是这样工作的。真正长期陪跑的 assistant，核心能力其实是“改得过来”，不是“第一次就看起来很聪明”。

## 一句话结论
今天最强的研究信号是，下一代 agent 和 model 的关键能力，越来越像 process intelligence，而不是 answer styling。对 coding model 来说，是先会在内部执行一遍；对 post-training 来说，是给不同样本不同学习信号；对 persistent agent 来说，是在变化的环境里持续修正自己。谁更会模拟过程，谁就更可能在真实世界里赢。