---
title: "Paper Digest: 2026-10-08"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Post-training, Evaluation, On-Policy Distillation]
---

今天最值得看的 paper 是 **Before They Can Solve: Predicting Post-Training Coding-Agent Performance from Base Models**。

它问了一个很贵的问题：准备训练 coding agent 时，怎样提前判断哪个 base checkpoint 值得投入一轮昂贵的 SFT 或 RL？

常用的 end-to-end pass@K 在这里并不可靠。很多 base model 已经对正确 patch 保留了足够的 probability mass，却还不会稳定地产生格式正确的 tool call，也无法从 cold start 驾驶完整 agent harness。最终分数把潜在的 coding capability 和尚未训练好的工具使用能力混在了一起。

## 从成功轨迹里找到 decisive action

论文利用已经完成 post-training 的 agent 成功轨迹。对每条轨迹，作者逐步 replay，并在每次代码修改后重新运行 tests。

第一次让 repository 从 failing 变成 passing 的代码动作，被定义为 decisive action。它有两个关键性质：

- 前面的 context 来自真实成功轨迹
- 当前 patch 的有效性由 repository tests 直接认证

这样，评估 base checkpoint 时不再要求它独立完成此前所有 tool calls。模型只需要在一个已经推进到关键节点的状态里，证明自己是否支持那个真正解决问题的动作。

## 三个低成本 screen

作者围绕 decisive step 设计了三种评估：

1. **Decisive-Action BPB**：计算 base model 给认证动作分配的 bits per byte，衡量它对完整关键动作的 likelihood。
2. **Patch MCQ**：让模型在正确 patch 和被同一 verifier 拒绝的 alternatives 之间选择。
3. **Prefix-conditioned pass@K**：给定成功轨迹的前缀继续生成，只要后续 patch 能通过 tests 就算成功，无需复现原轨迹的字面答案。

在十组公开 base model 和 post-trained model 配对上，这三个 screen 得到的 checkpoint 排名都与最终 SWE-bench Verified pass@1 高度一致。

这里最有价值的部分，是把 agentic post-training 的选型问题拆开了。Harness proficiency 可以通过训练快速改善，而 base distribution 是否已经覆盖关键 coding action，决定了后续训练有没有足够好的起点。

## 对 agentic post-training 的实际意义

一条可落地的 checkpoint selection pipeline 可以这样做：

1. 收集 benchmark 中 post-trained agent 的成功 trajectories
2. 在每个 code-changing step 后运行 verifier
3. 定位第一个让任务通过的 decisive action
4. 用 BPB、MCQ 和 prefix-conditioned sampling 筛选 base checkpoints
5. 只对排名靠前的候选投入完整 SFT/RL run

这套方法还留下一个更有意思的方向：decisive action 不必只用于 evaluation。它也可以成为 curriculum construction、trajectory filtering 或 training-data weighting 的信号。若某个 checkpoint 已经接近关键动作，可以安排更具挑战性的轨迹；若 likelihood 很低，则先补对应能力的数据。

## 今天另外两篇相关论文

**Gains and Collapse in On-Policy Distillation** 把 OPD 解释为 teacher 提供隐式 reward 的 RL。Teacher 可能偏好自己很少生成的异常行为，student 的 on-policy exploration 会找到这些漏洞，最终出现超长和重复输出。论文表明，过滤 unhealthy rollouts 与 SFT initialization 都能缓解 collapse。对 OPD pipeline 来说，teacher 的生成质量远远不够，还要审计它如何评价 student distribution 上的行为。

**When Sub-Agents Work in Parallel** 对 Codex、Claude Code 和 Kimi Code 做了 matched concurrency study。354 个任务、2,124 次执行覆盖不同复杂度和 horizon，并归纳了 13 类 concurrency-specific failure modes。它提醒 agent builder 把动态并发当作 scheduling policy 来测量：任务依赖、重复劳动、结果到达时机、integration cost 和 lead-agent attention 都会决定 parallel compute 是否真正转化为成功率。

论文：<https://arxiv.org/abs/2610.10478>

另外两篇：<https://arxiv.org/abs/2610.03185>、<https://arxiv.org/abs/2610.10263>
