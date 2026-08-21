---
title: "Paper Digest: 2026-08-21"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, Environment Engineering, Code Agents, Post-Training]
---

今天最值得看的 paper，我会选 **EnvHarness: Awakening Static Worlds for Agent Learning**。

Agent RL 有一个很朴素的矛盾。Policy 在持续更新，environment 却常常保持原样。训练初期有价值的任务，在模型掌握主要模式后会迅速失去区分度。Environment 也看不到 policy 当前卡在哪里，只能重复提供既定分布里的样本。

重新生成 environment 听起来合理，工程代价却很高。每个领域都有自己的状态、工具、任务逻辑和 verifier。生成器一旦改坏，reward 也可能随之失真。训练信号看似更难，实际可能只是引入了不可解任务或验证漏洞。

EnvHarness 的想法很干净：保留原始 environment 和 verifier，在外面增加一层可编程的 plug-in components。这个 wrapper 可以调整环境行为、针对 policy 弱点生成新挑战，同时继续使用已经被信任的验证逻辑。

## Environment 也需要 harness

今天的 agent system 通常已经有 model harness：prompt、tools、memory、control flow 和 retry policy 共同决定最终行为。EnvHarness 把同样的模块化思想放到 environment 一侧。

它提供一组标准接口，让 plug-in components 改变 agent 实际遇到的任务形态。底层环境逻辑保持完整，原有 verifier 仍然负责判断成功。这样做有两个直接好处：

- 不需要为每个新 curriculum 重写完整 environment
- 新任务继续继承原始 verifier 的可执行语义

这层抽象的价值，在于它把 environment engineering 变成可组合、可验证的系统工作。团队可以修改难度、状态暴露、任务条件或交互结构，同时把变化限制在明确的 wrapper 边界内。

## EnvRigger：根据 trajectory 找出当前弱点

EnvHarness 负责承载变化，EnvRigger 负责决定应该改变什么。

EnvRigger 把 target policy 当作 black box。它观察 execution trajectories，诊断反复出现的失败模式，然后合成对应的 EnvHarness components。新组件不会直接进入训练循环，它们需要经过 fresh rollouts 验证，确认自己确实产生了有意义的挑战。

这个流程形成一个 outer loop：

1. 在现有环境中运行 policy
2. 从 trajectory 中定位能力缺口
3. 合成针对性的 environment components
4. 用新 rollouts 验证组件质量
5. 把通过验证的环境交给 policy 继续学习

这里最关键的是 fresh-rollout validation。只根据同一批失败轨迹修改 environment，很容易过拟合某几个 episode。新的 rollout 能检查组件是否捕捉到稳定的能力缺口，以及它有没有破坏任务的可解性。

## 保留 verifier 是很强的工程约束

Agent training 中，environment generator 和 verifier 同时变化会带来严重的 credit assignment 问题。Policy 分数提高时，我们很难判断能力真的增强了，还是任务分布、reward definition 或验证漏洞发生了变化。

EnvHarness 要求 reshaped environment 保留原始 verifier。这个约束压缩了系统中的自由度，也让改动更容易审计。Environment component 可以创造新的训练压力，成功标准仍然由底层任务定义。

对于 terminal、web、code 或 tool-use agent，这一点尤其重要。Executable verifier 通常是最昂贵的资产之一。它凝结了任务状态、边界条件和正确性判断。能够复用 verifier，意味着已有 benchmark 可以成为持续训练的基础设施，而不会在每轮 curriculum 更新时重新承担验证风险。

## 实验结果说明了什么

论文在四个领域、五个 benchmarks 上测试 EnvHarness。相比原始环境和 domain-specific environment generation pipelines，它在 held-out instances 上最高提升 9.0 points，同时减少 9.8% execution steps。

这组结果值得一起看。更高成功率说明 reshaped environments 找到了有价值的训练区域，执行步骤下降则表明 policy 没有靠更长、更昂贵的搜索换取分数。训练信号更聚焦后，agent 可以用更短路径解决未见实例。

作者还发现，EnvHarness 提供了更好的 reinforcement learning optimization signal。这里的含义很实际：RL 的瓶颈经常落在数据分布上。大量 rollout 如果持续重复 policy 已经会做的事情，增加 compute 也不会带来足够梯度信息。针对当前弱点设计 environment，可以提高每次 rollout 的训练价值。

## Policy 和 environment 的双层优化

EnvRigger 可以被理解成一个 curriculum optimizer。Inner loop 更新 policy，outer loop 根据最新 trajectory 调整 environment。两层系统都围绕同一个目标工作，但更新频率、预算和稳定性要求并不相同。

这会带来几个值得继续研究的问题：

- policy 更新多快之后，旧的 weakness diagnosis 会失效
- rollout budget 应该怎样分配给训练、诊断和组件验证
- 多个 harness components 同时存在时，怎样避免它们相互干扰
- environment 变得太快时，policy optimization 是否会失去稳定目标
- 哪些弱点适合通过 curriculum 修复，哪些应该进入 SFT data 或 reward design

在大规模训练系统里，这些问题还会受到 actor-learner lag 和 heterogeneous workers 影响。EnvRigger 看到的 trajectory 可能来自旧 policy，组件验证又可能在另一批 workers 上完成。Outer-loop decision 必须带上 policy version、environment version 和 rollout provenance，才能知道一个 improvement signal 属于哪个系统状态。

## 对 post-training pipeline 的启发

这篇论文给 agent post-training 提供了一个很实用的分层：trajectory 不只用于更新 weights，也可以用来更新产生 trajectory 的环境。

一条失败轨迹至少能提供三类信号：

- policy error：模型缺少知识、规划或执行能力
- harness error：prompt、tools、memory 或 control flow 让模型无法发挥能力
- environment opportunity：当前任务可以被重组，用来更集中地训练暴露出的缺口

传统 pipeline 往往只消费第一类信号。EnvHarness 让第三类信号也能沉淀成可复用资产。一个诊断过的失败模式，可以对应一个经过验证的 environment component，后续 policy 版本仍然可以用它做 regression testing 或 targeted RL。

进一步看，这些 components 还可以形成 curriculum archive。系统记录每个组件针对的 failure mode、适用 policy range、验证结果和训练收益。新模型进入训练时，可以先跑 archive 中的诊断集，再决定哪些环境值得重新激活。

## 为什么今天选它

更强的 agent 需要高质量 trajectory，也需要持续产生高信息量 trajectory 的环境。静态 benchmark 很适合比较系统，却很难长期承担训练 curriculum。

EnvHarness 提供了一条工程上可信的路径：复用已有环境和 verifier，通过标准化 wrapper 注入针对性的变化，再用 fresh rollouts 检查这些变化。EnvRigger 则让 trajectory diagnosis 直接参与下一轮 environment design。

对于正在做 agent RL、synthetic task generation 或 harness optimization 的团队，这篇论文值得细读。它提醒我们，rollout compute 的价值取决于 policy 遇到了什么。Environment 能够持续对准当前弱点时，同样的训练预算会产生更有效的学习信号。

## 另外两篇

第二篇是 **FACET: Preserving Source Intent and Executable State in Terminal Task Synthesis**。

FACET 关注 terminal-agent synthetic data 的一致性。一个任务同时包含 instruction、initialized environment、reference solution 和 executable verifier，这些 artifacts 如果基于不同假设生成，任务可能不可解，或者 verifier 会判断错误。FACET 先构造并修复 container state，再让所有 artifacts 共享这个 executable grounding，并用执行验证与 targeted repair 修复局部问题。生成的成功 trajectories 在多个模型规模上都能改善 Terminal-Bench 2.1。对 code-agent data pipeline 来说，environment-first ordering 是一个很具体的设计原则。

第三篇是 **SWE-bench Science: Can Coding Agents Resolve Engineering Tasks in Science?**。

它收集了 98 个 GitHub repositories、20 个科学领域里的 119 个任务，覆盖 issue-driven repair、expert exploration 和 engineering integration。Claude Code with Opus-5 (max) 的 pass@1 仍低于 50%。论文识别出四类常见失败：科学知识或抽象不足、探索方向错误或修复停留在表面、repair coverage 与 system integration 不完整，以及科学知识无法泛化。更有意思的是 paired ablation：准确的 domain guidance 可以提升平均表现和 token efficiency，错配的 guidance 会形成 anchoring，却未必提高 exact repair success。这对 agent memory 和 domain RAG 都是一个重要警告。

论文：<https://arxiv.org/abs/2608.19880>

另外两篇：<https://arxiv.org/abs/2608.18580>、<https://arxiv.org/abs/2608.19799>
