---
title: "Paper Digest: 2026-10-15"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Multimodal, Reinforcement Learning, Post-training]
---

今天最值得看的 paper 是 **PrismaCoder: Pushing the Boundaries of Open Multimodal Code Intelligence through Decomposed Rubric Synthesis**。

Coding model 的评测长期依赖一个舒服的假设：程序只要能运行、测试只要能通过，结果就足够好。这个假设放到 visual coding 里会迅速失效。网页可能没有报错，却布局混乱；图表可以成功渲染，却看不清坐标和图例；生成的 slide 语法完全正确，视觉层级依然像一份未经整理的草稿。

PrismaCoder 处理的正是这个 verifier gap。它覆盖网页、图表、动画和半结构化文档等任务，把 code correctness、工程质量与 rendered output 的视觉效果放进同一条数据和训练 pipeline。作者据此构建了 PrismaCoder-2M，并训练、开源 PrismaCoder-9B 与 PrismaCoder-35B。论文报告，两款模型大幅刷新 open model 的 multimodal coding 表现，并接近领先的 proprietary systems。

## Visual coding 缺少怎样的 reward

传统代码任务拥有相对明确的 executable verifier。单元测试通过，至少说明输出满足了一部分行为契约。Visual coding 同时面对三组目标：

1. 程序能否正确执行。
2. 实现是否完整、合理、可维护。
3. 最终渲染结果是否真正符合设计意图。

现有单一信号很难覆盖三者。Execution result 通常只给出 pass 或 fail；code similarity 无法可靠预测视觉效果；为某一类任务写的 verifier 也很难迁移到网页、海报、图表和动画。

如果把所有要求压缩成一个整体分数，训练系统还会失去关键的 credit assignment。模型只知道这张网页得到 0.63，却不知道问题出在交互失效、信息缺失、对齐混乱，还是配色和可读性。这样的 reward 很容易让 policy 追逐 judge 偏好的表面风格。

## PrismaForge：先生成评测合同

论文的核心是 **PrismaForge**。它为每个任务合成一份 hierarchical rubric，把模糊的「做得好」拆成可以逐项检查的 criteria。

一份 rubric 可以分别要求页面包含哪些内容、哪些交互必须工作、布局关系如何组织、视觉重点落在哪里，以及 rendered artifact 应满足什么质量标准。每项要求都更接近可验证的局部合同，judge 可以针对具体证据评分，训练系统也能看到失分来自哪里。

这里最漂亮的设计，是同一份 rubric 被重复使用：

- 在 data synthesis 阶段，它规定样本需要满足的目标。
- 在 SFT filtering 阶段，它淘汰程序正确但视觉质量差的样本。
- 在 RL 阶段，它提供更细的 reward signal。
- 在 evaluation 阶段，它继续作为结构化评分标准。

数据过滤和 RL reward 因此共享同一种任务语义。Pipeline 不必在每个阶段重新发明一套 judge，也减少了训练目标与最终评测彼此错位的机会。

## 从 rubric 到 2M open dataset

作者利用 PrismaForge 构建了 **PrismaCoder-2M**，称其为目前规模最大、任务最多样的 open multimodal code intelligence dataset。随后，他们在这些数据上训练 9B 和 35B 模型，并让 synthesized rubric 继续参与 post-training。

这个结果的重要性超出了又一组 benchmark 分数。Open visual coding research 一直受制于高质量数据。收集普通代码相对容易，收集带有渲染结果、设计要求、执行状态和细粒度质量标签的代码昂贵得多。Rubric synthesis 提供了一种规模化制造 supervision 的方法，同时保留了任务级可解释性。

对于训练团队，一条可执行的数据链路大致是：

1. 从 task intent 生成 decomposed rubric。
2. 生成代码并在 sandbox 中执行。
3. 用 browser 或 renderer 获取最终 artifact。
4. 将执行证据、代码证据和视觉证据分别对应到 rubric criteria。
5. 用结构化得分过滤 SFT data。
6. 在 on-policy rollout 中复用 rubric 作为 reward。

## 最值得验证的四个问题

第一，rubric 的增益究竟来自更好的数据过滤，还是更细的 RL reward？这两部分需要在相同数据量和 rollout budget 下拆开验证。

第二，rubric judge 会不会形成新的 reward hacking surface？如果 rubric 反复强调对齐、留白和色彩协调，模型可能学会一套安全但同质化的审美捷径。高分 artifact 仍需接受真实用户任务和分布外设计风格的检验。

第三，criteria 之间如何聚合？程序无法运行时，漂亮的视觉草图是否还应获得部分 reward；核心交互缺失时，视觉质量又该占多大权重。Aggregation rule 实际上定义了 policy 的行为偏好。

第四，这条 pipeline 的 systems cost 很高。每个 sample 都可能涉及代码执行、browser rendering、截图、multimodal judging 和 artifact storage。大规模训练需要处理异构 worker、失败重试、去重、judge batching 与严格的数据 lineage。

对做 code model、SFT/RL 和 agent infrastructure 的团队来说，PrismaCoder 最有价值的想法很朴素：先把「好结果」拆成可检查的合同，再让数据生产和 policy optimization 围绕同一份合同工作。Visual coding 只是最明显的应用场景。任何结果质量无法被一个 unit test 覆盖的 agent task，都可能需要这样的 decomposed verifier。

## 今天另外两篇相关论文

**Doc2Harness: Benchmarking LLMs on Long-Horizon Harness Engineering of Coding Agents** 把评测对象放到 coding agent 的 harness 本身。Builder model 根据 interface contract 和一份汇总十二种 frontier harness 行为的开发文档，在隔离容器里构建完整实现。冻结后的 harness 再由 scripted mock model 驱动，并通过 328 个 deterministic cases 检查 model requests、files、exit code 和 command output。十九个 frontier models 中，最高 pass rate 只有 53.05%。这个 benchmark 说明，context construction、tool protocol、state 和 process control 这些基础设施能力，依然是 long-horizon agent 的主要短板。

**Beyond Domain-Routed Supervision: Cross-Teacher Guarding for Multi-Teacher On-Policy Distillation** 研究多个 specialist teachers 如何共同训练一个 shared student。Road MOPD 让 non-routed teachers 充当 cross-domain critics，用 teacher agreement 调整 token-level correction，并用 auxiliary gradients 约束可能伤害其他 domain 的更新。论文在 high-conflict setting 下让弱势 domain 提升超过 5 个百分点。对于 multi-domain post-training，它提醒我们：把样本路由给正确 teacher 仍不够，共享参数上的 gradient interaction 也需要被显式管理。

论文：<https://arxiv.org/abs/2610.20538>

另外两篇：<https://arxiv.org/abs/2610.20344>、<https://arxiv.org/abs/2610.13521>
