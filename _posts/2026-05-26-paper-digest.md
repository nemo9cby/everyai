---
title: "Paper Digest: 2026-05-26"
categories: [Paper Digest]
tags: [AI, Agents, Coding Agents, RL, Post-Training]
---

今天最值得看的 paper，我会给 **ECHO**。

这篇 paper 讲 terminal agents，但它真正抓住的是 agent RL 里一个经常被浪费掉的信号：环境返回。

CLI agent 每一步都在和一个小世界互动。模型输出命令，terminal 执行命令，然后返回 stdout、stderr、文件变化、日志、trace 和错误信息。标准 GRPO 类训练通常只把 action tokens 当作优化对象，再用最后的稀疏 outcome reward 更新策略。问题是，失败 rollout 里其实有大量证据，说明环境怎么响应、命令哪里错、下一步该如何修正。

ECHO 的做法很直接：保留 policy-gradient loss，同时加一个 auxiliary loss，让模型预测自己动作之后会看到的 environment observation tokens。换句话说，terminal 输出不只喂回上下文，还变成了 dense supervision。

这个设计的工程味很强，因为它不需要额外 rollout，也不需要人写 expert demonstration。ECHO 复用 GRPO 的 forward pass，把原本已经产生的 terminal stream 变成训练目标。

结果也够硬。在 TerminalBench-2.0 上，ECHO 把 Qwen3-8B 的 GRPO pass@1 从 2.70% 提到 5.17%，把 Qwen3-14B 从 5.17% 提到 10.79%。从 base Qwen3-8B 出发，它在 held-out terminal tasks 上接近 expert-SFT-then-GRPO 的效果，而且没有用 expert demonstrations。更有意思的是，某些设置下只靠 environment prediction loss，也能带来 verifier-free self-improvement。

这对 coding agent 很重要。真实 repo 任务里，最有价值的信息往往就藏在命令输出、测试失败、lint 报错、文件 diff 和日志里。ECHO 的提醒是：这些东西不只是下一轮 prompt 的材料，也可以是训练 agent world model 的监督信号。

今天另外两篇也值得一起看。

**CUA-Gym** 处理 computer-use agent 的 RLVR 数据问题。它用 generator agent 生成初始状态和 golden state，用 discriminator agent 写 reward function，再用 orchestrator 反复执行和修正，最后构造出 32,112 个 verified training tuples，覆盖 110 个环境。它延续了 OpenComputer / EnvFactory 那条线：agent 要变强，关键是环境和 reward 要做实。

**CoSPlay** 则偏 code generation 的 test-time scaling。它绕开 ground-truth unit tests 的依赖，让候选代码和自生成测试互相筛选、互相修正。代码和测试形成 execution matrix 后，系统用 pass-count、test refresh 和 output consensus 来选最终答案。对缺少可信测试的 code RLVR 和 coding-agent repair，这是很实用的思路。

把这三篇放在一起看，今天的主线很清楚：agent 训练正在从单纯优化答案，走向更充分地利用环境反馈。terminal 输出、可执行软件世界、自生成测试，都在变成训练和验证的基础设施。

所以今天先读 `2605.24517`。如果我们认真做 coding agent，terminal stream 这种免费监督信号不该被浪费。

## 今日 3 篇精选

### 1) ECHO: Terminal Agents Learn World Models for Free
- 链接: https://arxiv.org/abs/2605.24517
- 摘要速读: 在 GRPO 之外加入 environment-token prediction，让 terminal 输出成为 CLI agent 的 dense on-policy supervision。
- 为什么重要: 它把 coding/terminal agent 每次 rollout 里已有的 stdout、error、log 和 trace 变成了训练信号。

### 2) CUA-Gym: Scaling Verifiable Training Environments and Tasks for Computer-Use Agents
- 链接: https://arxiv.org/abs/2605.25624
- 摘要速读: 自动生成 computer-use agent 的任务、环境状态和 deterministic reward functions，构建 32,112 个 verified RLVR training tuples。
- 为什么重要: 它说明 GUI/computer-use agent 的训练瓶颈，核心在可执行环境和可验证 reward。

### 3) CoSPlay: Cooperative Self-Play at Test-Time with Self-Generated Code and Unit Test
- 链接: https://arxiv.org/abs/2605.23491
- 摘要速读: 让候选代码和自生成 unit tests 在 test-time 互相筛选、修正和更新，不依赖 ground-truth tests。
- 为什么重要: 它给 code generation 提供了一条更实用的 inference-time verification 路线。

## 一句话结论
今天最强的研究信号是：**agent 的环境反馈正在变成可训练资产。** `2605.24517` 值得先读。
