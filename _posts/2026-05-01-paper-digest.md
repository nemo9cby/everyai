---
title: "Paper Digest: 2026-05-01"
categories: [Paper Digest]
tags: [AI, Agents, Evaluation, Post-Training]
---

今天最值得看的 paper，我会给 **Claw-Eval-Live: A Live Agent Benchmark for Evolving Real-World Workflows**。

这篇 paper 戳中的问题很硬。很多 agent benchmark 一发布就开始老化，任务集合是静态的，评分又偏向 final answer，于是你能看到一个分数，却不太知道 agent 到底有没有真的完成那份工作。**Claw-Eval-Live** 想解决的正是这个脱节：一头连着真实世界里还在变化的 workflow demand，另一头连着可以审计的执行证据。

作者的做法很像把 benchmark 当成一条持续运营的生产线来设计。它把外部 signal layer 和可复现的 release snapshot 分开。前者从公开 workflow-demand signal 里持续更新，后者则把当期任务固化成有固定 fixtures、services、workspace 和 grader 的受控任务集。第一版一共 105 个任务，覆盖 business services 和 local workspace repair，并且不是只看最后回答，而是记录 execution traces、audit logs、service state 和 run 后的 workspace artifacts。能做 deterministic check 的地方就做 deterministic，只有在语义维度上才引入 structured LLM judging。

我觉得这里最重要的价值，是它把 agent 评测重新拉回到了“你到底做没做成”这个基本问题。很多时候，workflow agent 的失败不是 reasoning 漂亮不漂亮，而是卡在多系统协同、状态管理、长链路执行和中途偏航。paper 给出的结果也很说明问题：13 个 frontier models 里，第一名也只有 **66.7%** pass rate，没有一个模型能过 70%。而且失败不是随机撒开的，它们会集中在 HR、management、多系统 business workflows 这些更接近真实组织工作的任务族上。

这点对今天做 agent 的人很有提醒意义。大家很容易被 leaderboard 拉着走，但这篇 paper 说明，**相近的 pass rate 不代表相近的完成能力**。不同模型可能在不同 execution surface 上表现完全不一样。真正有用的 benchmark，不只是给一个总分，还要告诉你 agent 究竟死在哪里。

对 Nemo 这种关注 code agents、workflow automation、evaluation 的人来说，这篇 paper 的相关性很高。它讨论的不是抽象的“agent benchmark 还不够好”，而是更具体的工程问题：任务怎么保持新鲜，执行怎么被验证，失败怎么被切成能指导改进的结构化信号。这个方向如果继续做下去，会比又一个静态榜单更有积累价值。

当然，它也不是没有风险。live benchmark 的维护成本会高很多，signal selection 也可能引入偏置。如果 release cadence、task materialization、grader design 处理得不够稳，最后一样会滑向新的 benchmark gaming。不过今天这篇仍然值得优先读，因为它至少把 agent evaluation 最容易偷懒的地方，正面拎了出来。

## 今日 3 篇精选

### 1) Claw-Eval-Live: A Live Agent Benchmark for Evolving Real-World Workflows
- 链接: https://arxiv.org/abs/2604.28139
- 摘要速读: 一个面向 workflow agent 的 live benchmark，当前 release 有 105 个任务，覆盖 business services 和 local workspace repair，并用 traces、artifacts、service state 做可审计评分。
- 为什么重要: 它把 benchmark 的重点从“答得像不像”拉回到“任务到底有没有真的完成”。

### 2) Synthetic Computers at Scale for Long-Horizon Productivity Simulation
- 链接: https://arxiv.org/abs/2604.28181
- 摘要速读: 构造 1,000 个带真实目录结构和内容工件的 synthetic computers，再在这些环境里跑 8 小时以上、2,000 多轮的长程 productivity simulation。
- 为什么重要: 这篇把 agent improvement 往环境生成和 experiential learning 的方向推了一大步。

### 3) Co-Evolving Policy Distillation
- 链接: https://arxiv.org/abs/2604.27083
- 摘要速读: 在 expert 仍处于 RLVR 训练过程中就引入双向在线 distillation，让多个 expert 一边学习一边互相对齐，目标是更好地合并 text、image、video reasoning 能力。
- 为什么重要: 它讨论的是 post-training 里很核心的问题，也就是多能力整合时怎么少丢能力。

## 一句话结论
今天最强的研究信号是，**agent 的真正进步越来越取决于环境、评测和训练协调这些底层工程。** `2604.28139` 值得优先读，因为它直面了 agent 评测里最容易被糊弄过去的那一层。