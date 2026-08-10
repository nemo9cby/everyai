---
title: "Paper Digest: 2026-08-10"
categories: [Paper Digest]
tags: [AI, Code Agents, Software Engineering, Planning, Fine-Tuning, Agent Scaffolds]
---

今天最值得看的 paper，我会选 **DCAS: Decoupling CLI Agent Scaffolding to Internalize Planning across Scaffolds**。

它指出 code-agent training 里一个容易被忽略的 confound：模型学到的内容同时包含解决软件问题的能力，以及生成 trajectory 的 agent scaffold 所规定的行为习惯。

当前开源 software-engineering agent 的训练数据高度集中在 OpenHands。用这些 trajectories fine-tune 的模型放回 OpenHands 时表现很好，换到其他 CLI scaffold 后却明显下降。论文还观察到，未经 fine-tuning 的 base model 没有同样明显的分化。这说明问题来自 fine-tuning distribution，scaffold conventions 已经进入模型参数。

## Scaffold 也是训练分布的一部分

CLI agent scaffold 决定很多看似外围、实际会影响 policy 的细节：

- 什么时候先写 plan
- plan 以什么格式出现
- tool call 与自然语言怎样交错
- 每轮如何读取 observation、更新状态并选择下一步
- 失败后是继续修补、重新规划，还是重启执行

如果所有训练 trajectories 都由同一种 scaffold 产生，模型很容易把这套固定 grammar 当成任务本身的一部分。此时同 scaffold evaluation 测到的是 model 与 harness 的组合能力，难以判断多少能力真正留在模型内部。

DCAS 把 planning 分为两个层面。

**Explicit planning** 是执行前生成的一份 first-class plan。它可以被单独替换、评分或干预。

**Implicit planning** 是贯穿 agent loop 的结构习惯，例如怎样分解动作、什么时候检查结果、如何在 observation 后调整策略。它通常隐藏在整条 trajectory 的组织方式里。

这个区分很有用。仅仅要求模型在开头输出一份 plan，并不保证它在后续执行中具备稳定的 planning behavior。反过来，一个不显式展示计划的 agent，也可能通过良好的行动结构表现出很强的 implicit planning。

## DCAS 做了什么

论文提出 Decoupling CLI Agent Scaffolding（DCAS），在 CLI scaffold 与 backend model 之间加入一个 API interception layer。它可以把任意 scaffold 发出的请求路由到指定 backend model，无需修改 scaffold 本身。

这层解耦支持两类关键实验。

第一类是 cross-scaffold evaluation：固定同一个模型，换不同的 CLI agent harness，直接测量 performance gap。

第二类是 planning-aware trajectory collection：控制 plan 的来源与执行 scaffold，把 explicit planning 和 implicit planning 对结果的贡献拆开。

论文通过 plan-source intervention 发现，planning quality 是高杠杆因素，它带来的增益可以超过观测到的 cross-scaffold drop。随后，作者只使用一小批 DCAS 收集的 planning-aware trajectories，在单一 scaffold 下 fine-tune 模型，却能让它在多个未参与训练的 scaffolds 上一致提升。

这支持一个很实用的判断：planning data 的结构质量可能比单纯扩大同质 trajectory 数量更重要。训练数据需要让模型内化可迁移的 planning capability，同时减少对某个 harness 固定 conventions 的依赖。

## 对 code-agent post-training 的启发

这篇 paper 给 code-agent pipeline 提供了几条直接可执行的检查项。

首先，evaluation matrix 应该同时包含 in-scaffold 与 cross-scaffold results。只在训练 scaffold 内测 benchmark，容易把 orchestration assistance 算到模型能力头上。

其次，trajectory dataset 需要记录 scaffold provenance。模型、task、reward、tool schema 之外，还应该记录 plan format、interaction protocol、retry policy 和 context construction。它们都可能成为隐藏的 domain label。

第三，data ablation 应该把 explicit plan 与 execution structure 分开。可以固定 execution trajectory，只替换 plan source；也可以固定 plan quality，改变 scaffold 的隐式交互规则。这样才能知道模型究竟学会了什么。

最后，scaffold diversity 可以成为 training curriculum 的一个维度。同一个 task 由多个 harness 采集 trajectories，可能帮助模型识别跨 scaffold 保持稳定的 task-solving invariants。需要进一步验证的是：多少种 scaffold 足够、不同 model families 是否共享这种收益，以及效果能否扩展到 repository-scale tasks。

## 为什么今天选它

DCAS 讨论的是一个基础评测问题：当 agent 的表现来自 model、prompt、tool interface、memory 和 control loop 的共同作用时，我们怎样确认 fine-tuning 真正提升了模型？

它给出的价值既有方法论，也有工程入口。Cross-scaffold testing 揭示 hidden overfitting，API interception 让控制实验可落地，planning-aware trajectories 则提供了一条改进数据的路线。

对于 code agents，这类工作会影响 benchmark 解读、trajectory collection、SFT recipe 和部署选择。模型在一个 scaffold 里拿到高分只是起点，能否把 planning 带到陌生 harness，才更接近可复用的能力。

## 另外两篇

第二篇是 **SFT Conflicts, RL Coexists: A Theoretical and Empirical Analysis of Multi-Task Learning for LLMs**。

论文比较 multi-task learning 中 SFT 与 RL 的 gradient interference。作者报告 SFT 在多阶段训练中容易发生任务冲突，RL 的更新则更稀疏、近似正交。理论分析把 SFT interference 描述为由 gradient norm 限制，把 RL interference 描述为受 advantage normalization 与 on-policy optimization 所产生的 gradient variance 限制。基于这一点，作者提出 Parallel-RL，允许不同任务的训练过程解耦执行。

第三篇是 **The Optimizer Is the Agent: Reasoning-Driven Search across Prompts, Programs, and ML Workflows**。

ReASearch 用同一个 tool-using agent loop 优化 prompts、programs 和 ML workflows。Agent 自己决定评估什么、怎样诊断失败、何时修改、验证或重启，并用 persistent memory 维护长程搜索状态。论文在 14 个任务上报告相对强 specialized baselines 的 2% 到 40% 提升。它最有意思的假设是，搜索控制器的一部分可以由 agent reasoning 直接承担。

论文：<https://arxiv.org/abs/2608.06113>

另外两篇：<https://arxiv.org/abs/2608.03573>、<https://arxiv.org/abs/2608.06714>
