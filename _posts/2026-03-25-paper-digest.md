---
title: "Paper Digest: 2026-03-25"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天 paper 这条线最有意思的，不是哪个模型又刷了一个分，而是 agent research 越来越像严肃的 systems engineering。大家开始更认真地处理 latency、workflow、coordination 这些脏问题。这个方向不花哨，但更接近产品到底能不能跑起来。

## 今日 3 篇精选

### 1) SpecEyes: Accelerating Agentic Multimodal LLMs via Speculative Perception and Planning
- 链接: https://arxiv.org/abs/2603.23483
- 摘要速读: 这篇盯住了 agentic multimodal system 里一个很现实的问题。视觉理解、推理、工具调用一旦串起来，延迟会迅速堆高，吞吐也会掉下来。SpecEyes 的核心思路，是先让一个更轻量、不调工具的小模型去做 speculative planning，提前猜执行轨迹，尽量早地终止昂贵的 tool chain，同时把一部分并行机会榨出来。
- 为什么重要: 这篇 paper 的价值，不只是“快了 1.1 到 3.35 倍”这个结果，而是它把 agent inference 看成一个 scheduling 问题。很多时候瓶颈不在模型会不会想，而在整条执行链太串行、太重、太难并发。如果这个方向成立，下一波 agent 提升会更多来自 systems design，而不是单纯拉长 reasoning trace。

### 2) Effective Strategies for Asynchronous Software Engineering Agents
- 链接: https://arxiv.org/abs/2603.21489
- 摘要速读: 这篇研究的是 long-horizon software engineering task 里，多 agent 异步协作到底怎样才靠谱。作者提出 CAID，把 centralized delegation、asynchronous execution、isolated workspaces 和 executable verification 绑在一起。说白了，就是别让一堆 agent 在同一个工作区里互相踩脚，而是让它们像成熟软件团队一样，分支隔离、依赖感知、最后再 merge。
- 为什么重要: 这类 paper 可贵的地方，在于它没有停在“多开几个 agent 就更强”的口号层面。真正困难的是 coordination，谁先做、谁依赖谁、合并时怎么验证。对 code agent 来说，这比再多一个 benchmark 小数点更接近现实。

### 3) From Static Templates to Dynamic Runtime Graphs: A Survey of Workflow Optimization for LLM Agents
- 链接: https://arxiv.org/abs/2603.22386
- 摘要速读: 这是一篇 survey，但选题很准。它把 agent system 重新表述成 agentic computation graphs，讨论 workflow 的结构是什么时候决定的，优化的是哪一部分，用什么 signal 来优化。核心贡献不是提出新方法，而是给这个越来越混乱的领域补一套更清楚的语言。
- 为什么重要: 现在很多 agent 讨论的问题，不是方法太少，而是概念太糊。template、runtime graph、execution trace 常常被混着讲。这篇 survey 的意义，在于它帮大家把这些层次拆开，至少先把问题说清楚。

## 可能感兴趣

### 4) WildWorld: A Large-Scale Dataset for Dynamic World Modeling with Actions and Explicit State toward Generative ARPG
- 链接: https://arxiv.org/abs/2603.23497
- 亮点: 世界模型数据集做得很大，而且显式标注 state 和 action。如果后面要聊 state-aware world modeling，这篇可以补位。

### 5) On the Direction of RLVR Updates for LLM Reasoning: Identification and Exploitation
- 链接: https://arxiv.org/abs/2603.22117
- 亮点: 从 update direction 解释 RLVR，而不只是看 update magnitude。更偏 thesis，如果想写硬一点的 post-training 判断，这篇值得继续跟。

## 一句话结论
今天最强的研究信号是，agent 这条线终于开始正视 execution cost 和 coordination cost。真正会拉开差距的，不只是“模型更会想”，而是谁能把整个 agent workflow 变得更快、更稳、更可控。
