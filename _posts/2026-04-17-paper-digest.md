---
title: "Paper Digest: 2026-04-17"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的 paper，是一篇把 **SWE agent 为什么总在长任务里失速** 说得很透的工作。**SWE-TRACE: Optimizing Long-Horizon SWE Agents Through Rubric Process Reward Models and Heuristic Test-Time Scaling** 关心的不是某个 flashy demo，也不是再堆一点 benchmark trick，而是 software engineering agent 这件事到底怎样才能在长轨迹里少浪费 token、少 reward hacking、少靠暴力搜索硬撞。

这篇 paper 的野心挺明确，它不只改一个环节，而是把 **数据、训练、推理** 三层一起重做。作者先构建一个偏向 shortest-path、token-efficient 轨迹的 60K SFT 数据集，然后在 RL 阶段引入 rubric-based process reward model，让模型在中间步骤就拿到更细粒度的反馈，不必只盯着最后 task 有没有过。最后，他们把同一套 rubric 继续拿到 test-time，用来做 action pruning，让 test-time scaling 变成一种被启发式引导的搜索，而不是简单粗暴地多采样、多并行、多烧钱。

这件事为什么重要，因为现在很多 SWE agent 的失败都很像同一个病。轨迹又长又肿，reward 太稀，search 成本太高，最后系统看起来像在做复杂推理，实际上很多预算都死在低效探索和错误分支上。SWE-TRACE 最有价值的地方，在于它承认这不是单点问题，而是一个整套 agent lifecycle 的优化问题。

## 今日 3 篇精选

### 1) SWE-TRACE: Optimizing Long-Horizon SWE Agents Through Rubric Process Reward Models and Heuristic Test-Time Scaling
- 链接: https://arxiv.org/abs/2604.14820
- 摘要速读: 作者提出一个面向长程 SWE 任务的统一框架。先用 stepwise oracle verification 蒸馏出更短、更省 token 的 SFT 轨迹，再用 rubric-based PRM 给中间过程提供 dense feedback，最后把这套 PRM 复用到 test-time 做候选动作裁剪。重点不只是 resolution rate 提升，而是把训练和推理都绑到同一套过程性标准上。
- 为什么重要: 这篇 paper 很像在回答一个现实问题，SWE agent 要真变强，靠更大模型够不够。作者给出的答案偏系统派，远远不够。你得同时优化 trajectory quality、reward shape 和 inference search，不然 agent 很容易在长任务里把预算烧空。

### 2) How to Fine-Tune a Reasoning Model? A Teacher-Student Cooperation Framework to Synthesize Student-Consistent SFT Data
- 链接: https://arxiv.org/abs/2604.14164
- 摘要速读: 这篇工作指出 synthetic SFT 的一个常见坑，teacher 很强，不代表 teacher 生成的数据适合 student。作者提出 TESSY，让 teacher 和 student 交替生成 token，使合成数据既保留 teacher 的 reasoning 能力，也维持 student 的风格分布。在 code generation 任务上，naive teacher-generated data 会把 Qwen3-8B 训差，TESSY 则明显改善。
- 为什么重要: 做 post-training 的人都该认真看这篇。它提醒人，数据质量不只是对不对，也包括和 student distribution 合不合。

### 3) DR^3-Eval: Towards Realistic and Reproducible Deep Research Evaluation
- 链接: https://arxiv.org/abs/2604.14683
- 摘要速读: DR^3-Eval 试图给 deep research agent 建一个更真实、又可复现的 benchmark。任务基于静态 research sandbox，但保留支持材料、干扰项和噪声，同时用 recall、accuracy、citation coverage、instruction following、depth quality 等多个维度打分。
- 为什么重要: research agent 现在最缺的，往往不是 demo，而是可信评估。没有像样的 evaluation，很多“会做研究”都只是看起来像。

## 一句话结论
今天最强的研究信号是，agent 的进步越来越像整套系统工程。`2604.14820` 最值得看，因为它抓住了 SWE agent 最真实的痛点，长轨迹、稀疏奖励、昂贵搜索，并且试图用同一套 rubric 贯通训练和推理。这种味道，比单纯追 leaderboard 更接近下一阶段真正能落地的东西。