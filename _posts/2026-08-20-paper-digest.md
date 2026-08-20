---
title: "Paper Digest: 2026-08-20"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, Post-Training, Self-Play, Code Agents]
---

今天最值得看的 paper，我会选 **SPADE: Self-Play in Adaptive Synthetic Executable Environments**。

Agent RL 有一个很现实的上限：模型可以持续训练，环境池却常常停在原地。人工任务、静态合成任务和固定 verifier 都会逐渐被学会。继续增加 rollout，只是在重复已经缺少学习信号的分布。

SPADE 把 environment design 也放进 self-play。一个 LLM 同时承担两个角色：Environment Designer 编写完整的可执行训练环境，Reasoning Agent 在这些环境中行动并接受 RL。两者共同构成一条能够持续制造新挑战的训练回路。

## 环境本身就是代码

Environment Designer 输出的是带有 `reset()` 和 `step()` 接口的状态化程序。环境包含状态转移、reward function、verification code 和多轮交互逻辑。同一套接口可以描述 reasoning problem，也可以描述需要多步工具调用的 agent task。

这个设计让任务生成获得了几个重要属性：

- 环境可以真正执行，reward 和 state transition 能被程序检查
- 任务可以包含长期状态，超出单轮问答的边界
- Designer 可以同时改变 goal、dynamics 和 verifier
- 训练系统可以用统一 sandbox infrastructure 承载多类任务

对于 code agent 和 tool-use agent，这比生成一批自然语言题目更接近真实 rollout。模型面对的是一个会响应 action 的环境，最终结果由代码验证。

## 用 privileged hint 构造 regret

SPADE 需要判断什么样的环境最值得生成。太简单的任务没有梯度，完全无法完成的任务也难以提供有效学习信号。

论文使用 privileged hints 估计 regret。同一个环境中，Reasoning Agent 分别在有提示和无提示条件下尝试。两者的 reward gap 越大，说明任务位于一个有价值的区域：当前 agent 具备潜在解题能力，但仍缺少独立完成所需的策略或推理。

Environment Designer 因此会寻找 agent 的当前能力边界。随着 solver 进步，最高 regret 的任务分布也会变化，curriculum 可以随训练状态自适应更新。

这个信号很有吸引力，因为它同时约束 challenge 和 feasibility。Designer 需要制造能够暴露能力缺口的环境，也需要保证 privileged information 足以帮助 agent 跨过缺口。

## 两个容易被忽略的工程组件

作者发现，Environment Designer 需要从大规模 pretraining corpus 采样文档作为 grounding。缺少外部语义素材时，生成环境容易收缩到有限模式。

第二个关键组件是 accumulated environment memory。Designer 需要记住此前生成过什么、哪些任务有效、哪些模式已经被 solver 掌握。环境记忆承担了跨 iteration 的经验积累，帮助系统维持多样性并继续探索新的能力边界。

这两个结果说明，open-ended self-play 依赖的不只有 RL objective。Designer 的知识来源和长期记忆会直接决定生成分布的宽度。

## 实验结果

SPADE 扩展到 30B 参数模型。在八个 held-out math、science、code 和 reasoning benchmarks 上，相比最强 fixed-environment baseline 平均提升 5.3 points。

Agent tool-use 场景的提升更明显：

- BFCL-v4 multi-turn 提升 5.7 points
- ACEBench-Agent 提升 13.9 points
- Games setting 中，相对最强 baseline 的优势随模型规模增加

Held-out 结果很重要。它表明 Designer 生成的训练经验具有一定迁移性，收益没有完全锁在它自己写出的环境中。

## 对 post-training system 的启发

SPADE 给出了一种新的 scaling dimension：除了扩大 model、rollout batch 和 static task pool，还可以训练一个持续生成 executable environments 的 policy。

真正落地时，几个系统问题会很快出现：

- 生成代码的 sandbox isolation 与 resource budget
- Designer 和 solver 更新速度不同导致的 regret staleness
- Environment mode collapse 与重复任务检测
- Verifier gaming 和错误 reward function
- 长短环境混合时的 rollout scheduling
- 两个 policy 共享或分离参数时的训练稳定性
- 分布式 actor 中 environment memory 的一致性与更新延迟

这些问题很适合放进异构 rollout infrastructure 研究。Environment generation、execution 和 policy optimization 的资源需求不同，调度器需要处理长尾任务、失败 sandbox、动态 curriculum 和不断变化的 reward distribution。

对 code agent 还有一个直接扩展：让 Designer 根据真实 repository 文档、API、测试和 issue history 生成 project-specific executable environments，再把 solver 的能力迁移到 held-out repositories。这样可以同时测量 environment diversity、repository transfer 和 verifier robustness。

## 为什么今天选它

SPADE 的价值在于它提供了一个完整训练对象。Environment Designer 输出可执行程序，privileged-hint regret 定位能力边界，document grounding 扩展任务语义，environment memory 保存探索历史，Reasoning Agent 则在这些动态环境中持续学习。

这套结构把 adaptive curriculum、self-play 和 agent infrastructure 连在了一起。对研究 open-ended post-training 的团队，它值得细读。

## 另外两篇

第二篇是 **SkillGate: Training In-Policy Skill Selection in Long-Horizon Agents**。

SkillGate 研究 long-horizon agent 中一个很具体的 credit assignment 问题。Agent 选择 skill 时通常只生成少量 skill-name tokens，传统 sequence-level advantage 会把最终结果广播给整条 trajectory。后续执行失败时，一次正确的 skill selection 也可能收到负梯度。SkillGate 把 credit 分成两个 disjoint channels：outcome credit 只作用于 execution tokens，action-local advantage 精确作用于 skill selection tokens。在五个 agent benchmarks 上，它把 9B policy 的 trial success 从 40.8% 提高到 53.2%。这对 OpenClaw、Codex 和大规模 skill library 都很直接。

第三篇是 **SkillForge: Self-Distilling Agents for Project-Specific Issue Resolution**。

SkillForge 让 coding agent 在真实 issue 到来前主动学习 repository。系统根据 test-covered core functionality 合成 project-specific issues，再通过解决这些问题提炼 entity-grounded skills，并绑定到相关 repository entities。这样可以把一次探索沉淀为后续任务可复用的项目知识。它与 SkillGate 很互补：前者负责生成和组织 skills，后者负责训练 policy 在 trajectory 中选对 skill。

论文：<https://arxiv.org/abs/2608.19197>

另外两篇：<https://arxiv.org/abs/2608.18852>、<https://arxiv.org/abs/2608.18933>
