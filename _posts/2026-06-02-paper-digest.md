---
title: "Paper Digest: 2026-06-02"
categories: [Paper Digest]
tags: [AI, Agents, Search Agents, Coding Agents, Post-Training, Reinforcement Learning]
---

今天最值得看的 paper，我会给 **Harness-1: Reinforcement Learning for Search Agents with State-Externalizing Harnesses**。

这篇 paper 讲 search agent，但真正有价值的地方是它重新画了一条边界：哪些事情应该交给模型学，哪些事情应该交给 agent harness 维护。

很多 search agent 会把所有东西塞进不断增长的 transcript。模型要决定下一步搜什么，也要记住已经看过什么、哪些证据有用、哪些约束还没关闭、哪些 claim 已经验证过。这样做的问题很直接：RL 在优化搜索策略的同时，还要优化一堆可恢复的 bookkeeping。

Harness-1 的选择是把这些 routine state 外置。它做了一个 stateful search harness，维护 candidate pool、importance-tagged curated set、compact evidence links、verification records、compressed observations 和 budget-aware context rendering。模型不用把这些状态全靠上下文硬背，它专注于更语义化的动作：搜什么、保留什么、丢弃什么、验证什么、什么时候停止。

这个设计对 agent training 很关键。长程 agent 的很多失败并不来自模型完全不会推理，而是来自状态管理逐渐变脏：证据重复、上下文膨胀、claim 没被验证、旧 observation 占据 token budget、模型在 transcript 里迷路。Harness-1 把这些状态变成环境的一部分，让训练信号落在更清楚的动作边界上。

结果也不错。Harness-1 是一个 20B retrieval subagent，在 8 个 retrieval benchmarks 上达到 0.730 average curated recall，比下一个最强 open search subagent 高 11.4 个点，并且在 held-out transfer benchmarks 上表现尤其强。这个 transfer 结果是我最在意的部分，因为它说明 RL over explicit search state 可能学到的是可迁移的 retrieval behavior，而不是只记住训练域里的搜索套路。

这篇和昨天的 GrepSeek 很适合连着看。GrepSeek 把 corpus 变成可执行环境，让 search agent 直接用 shell command 互动。Harness-1 进一步强调：环境不只是工具集合，它还可以持有工作记忆、证据链接、验证记录和上下文渲染策略。

对 coding agent 来说，这个方向更值得认真拆。真实 repo 任务里，agent 一边用 `rg`、test、logs 和文件阅读定位问题，一边维护假设、证据和待验证项。如果所有状态都只靠 prompt 累积，系统会很快变贵、变乱、变脆。更合理的 agent runtime 应该把 workspace state、evidence state、verification state 显式化，然后让模型学决策。

今天第二篇是 **BenchEvolver: Frontier Task Synthesis via Solution-Centric Evolution**。它处理 code RLVR 的任务供给问题。现在很多 coding benchmark 已经被 frontier models 刷到很高分，容易题没有足够训练信号。BenchEvolver 不从零生成题目，而是先 evolution reference solutions，再从 evolved solutions 反推 statements 和 tests。这样生成出的任务有 executable semantics，也更容易验证正确性。

作者把它用在 LiveCodeBench 和 SciCode 上，构造了更难的变体，还整理出 LiveCodeBench-Plus。更重要的是，RL on evolved tasks 给 gpt-oss-20b 带来 held-out coding performance gains。这说明它不只是一个更难的榜单，也可能成为 code RLVR 的数据发动机。

第三篇是 **SABER: Benchmarking Operational Safety of LLM Coding Agents in Stateful Project Workspaces**。它把 coding-agent safety 从 isolated response 拉回到真实 workspace。SABER 看的是 agent 一串动作之后最终环境状态是否安全，并按 violation cause 做分类。结果很刺眼：即使表现最好的模型也有超过 54% harmful safety-violation rate。

这对生产系统很现实。一个 coding agent 的安全性不能只看它会不会拒绝危险 prompt。只要它能改文件、跑命令、移动项目状态，真正需要评估的是 action sequence 和 final workspace state。

三篇放在一起看，今天的信号很集中：agent 研究正在把环境状态显式化。Search agent 需要 harness memory，code RLVR 需要 executable evolved tasks，coding-agent safety 需要 final workspace state。

所以今天先读 `2606.02373`。如果你关心 search agent、repo agent、RL post-training、或者 agent runtime 设计，这篇是最值得拆的。

## 今日 3 篇精选

### 1) Harness-1: Reinforcement Learning for Search Agents with State-Externalizing Harnesses
- 链接: https://arxiv.org/abs/2606.02373
- 摘要速读: 在 stateful search harness 里训练 20B search agent，把 candidate pool、evidence links、verification records 和 context rendering 交给环境维护。
- 为什么重要: 它把 agent 训练的状态边界讲清楚了，模型学搜索决策，harness 管可恢复的工作记忆。

### 2) BenchEvolver: Frontier Task Synthesis via Solution-Centric Evolution
- 链接: https://arxiv.org/abs/2606.01286
- 摘要速读: 通过演化 reference solutions，再生成对应题面和 tests，把已有 coding problems 变成更难、更可验证的训练任务。
- 为什么重要: Code RLVR 的核心瓶颈之一是 near-frontier verifiable tasks，这篇给了一个可扩展的数据生成方向。

### 3) SABER: Benchmarking Operational Safety of LLM Coding Agents in Stateful Project Workspaces
- 链接: https://arxiv.org/abs/2606.01317
- 摘要速读: 在真实项目 workspace 中评估 coding agent 的 action sequence，按最终环境状态判断 operational safety。
- 为什么重要: coding-agent safety 必须覆盖文件、命令和项目状态变化，单轮回答安全远远不够。

## 一句话结论
今天最强的研究信号是：**agent 能力和安全都开始围绕显式环境状态重新设计。** `2606.02373` 值得先读。
