---
title: "Paper Digest: 2026-06-03"
categories: [Paper Digest]
tags: [AI, Post-Training, Distillation, Reinforcement Learning, Agents, Code Generation]
---

今天最值得看的 paper，我会给 **Trust Region On-Policy Distillation**。

这篇 paper 处理的是一个很实际的 post-training 问题：我们想用更强的 teacher model 带一个 student model，让 student 在自己的 generation trajectory 上学习。这个方向叫 On-Policy Distillation，简称 OPD。

OPD 介于 SFT 和 RL 之间。SFT 用离线数据，容易碰到 distribution shift。RL 可以在模型自己的轨迹上优化，但 reward 稀疏、训练复杂、credit assignment 难。OPD 的吸引力在于：让 student 自己生成，再用 teacher 给 token-level supervision。理论上，它既有 on-policy 的分布匹配，又有 dense supervision 的训练效率。

问题也在这里。student 自己生成的 token 不一定落在 teacher 熟悉、可靠的区域里。一旦 teacher 和 student 的分布差得比较远，teacher 对 student-generated tokens 的监督信号可能很脏。你以为是在蒸馏能力，实际可能是在把不可靠的梯度灌进模型里，严重时会导致 optimization failure。

TrOPD 的核心思路很直接：只在 teacher signal 可信的区域做 OPD。

它把 student trajectory 分成更可靠的 trust-region 和更危险的 outlier region。对 trust-region，继续用 reverse-KL 风格的 token supervision。对 outlier region，则尝试 gradient clipping、masking、forward-KL estimation 等更稳的处理方式。除此之外，它还加入 off-policy guidance：让 student 从 teacher prefixes 继续生成，再用 forward KL 做 imitation，引导 student 往更可靠的区域探索。

这个设计的价值在于，它承认 teacher supervision 不是全局同质的。teacher 在某些 token 上很有指导意义，在另一些 token 上可能已经偏离得太远。训练策略应该区分这些情况，而不是把所有 on-policy token 都当成同样可靠的学习对象。

实验上，TrOPD 在 mathematical reasoning、code generation 和 general-domain benchmarks 上都超过了 OPD、EOPD、REOPOLD 等 baseline。对我来说，code generation 这个评估点很重要，因为它说明这不是只在数学题上调出来的技巧，而是可能影响更广泛的能力迁移。

这篇也和最近一段时间的 post-training 趋势很贴。很多团队都在寻找 SFT 和 RL 之间的中间路线：既不想完全依赖离线 traces，也不想把所有能力学习都交给 sparse reward。OPD 是一个很自然的中间层，而 TrOPD 进一步把稳定性问题摆上台面。

今天第二篇是 **A Local Perturbation Theory for Cross-Domain Interference and Recovery in Multi-Domain RL**。它讨论 multi-domain RL 里很常见的现象：一个 domain 训好了，另一个 domain 掉了。

作者认为，干扰不完全来自 catastrophic forgetting，也不能只看 global gradient conflict。不同 domain 的更新可能很 sparse，但会经过共享的 active computation routes。在这些共享路径上，更新方向会形成局部冲突。论文用 local perturbation model 解释这种低维 conflict subspace，并展示短暂的 domain refresh 可以恢复受损能力。例如 Code 到 Math 到 QA 到 Creative Writing 之后，再做一个很短的 Re-Math refresh，可以把 Math 从 57.66 拉回 66.04，同时基本保住其他 domain。

这篇对实际训练很有用。多能力模型最难的地方往往不是单点 SOTA，而是怎么让 math、code、QA、writing、agentic behavior 同时留在模型里。它提醒我们：平均指标会掩盖局部损伤，post-training pipeline 应该持续监控 domain-specific regression，并保留 targeted recovery 的手段。

第三篇是 **Adaptive Auto-Harness: Sustained Self-Improvement for Agentic System Deployment on Open-Ended Task Streams**。它关心的是 agent harness 的长期部署问题。

很多 auto-harness 工作会优化 prompts、tools、skills、memories 和 supporting infrastructure，但评估通常发生在固定 benchmark 上。真实系统完全不同：历史越来越长，任务分布会变，不同任务需要不同 harness，人也会在中间介入。Adaptive Auto-Harness 用 stateful multi-agent evolver、harness tree、solve-time routing 和 human-steering hooks 来处理这种开放任务流。

这个方向对 agent runtime 很有启发。一个长期运行的 coding agent 或 research agent，不应该只有一套不断膨胀的万能 prompt。更合理的做法是维护多个 harness，根据任务路由，并允许人类在历史信号不足的时候进行 steering。

三篇放在一起看，今天的信号很集中：post-training 和 agent system 都在变得更 reliability-aware。蒸馏需要 trust region，多领域 RL 需要局部恢复，agent harness 需要长期适配和路由。

所以今天先读 `2606.01249`。如果你关心 LLM post-training、teacher-student distillation、code generation capability transfer，或者 SFT 和 RL 之间的训练路线，这篇值得拆。

## 今日 3 篇精选

### 1) Trust Region On-Policy Distillation
- 链接: https://arxiv.org/abs/2606.01249
- 摘要速读: 提出 TrOPD，在 teacher signal 更可靠的 trust-region 做 on-policy distillation，对 outlier region 用 clipping、masking、forward-KL 等方式降低不可靠监督的伤害。
- 为什么重要: OPD 是 SFT 和 RL 之间很有潜力的 post-training 路线，这篇把 distribution mismatch 下的稳定性问题讲清楚了。

### 2) A Local Perturbation Theory for Cross-Domain Interference and Recovery in Multi-Domain RL
- 链接: https://arxiv.org/abs/2606.02398
- 摘要速读: 解释为什么多领域 RL 会出现一个能力涨、另一个能力掉，并提出短 refresh 或稀疏 rollback 这类局部恢复方法。
- 为什么重要: 真实 post-training 很少只训一个 domain，多能力共存才是难点。

### 3) Adaptive Auto-Harness: Sustained Self-Improvement for Agentic System Deployment on Open-Ended Task Streams
- 链接: https://arxiv.org/abs/2606.01770
- 摘要速读: 针对开放任务流里的 agent harness 持续优化，使用 harness tree、solve-time routing 和 human-steering hooks。
- 为什么重要: 长期运行的 agent 系统需要多 harness 路由和持续适配，单一万能配置很容易变脆。

## 一句话结论
今天最强的研究信号是：**post-training 和 agent runtime 都开始围绕可靠性重新设计。** `2606.01249` 值得先读。
