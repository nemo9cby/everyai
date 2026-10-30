---
title: "Paper Digest: 2026-10-30"
categories: [Paper Digest]
tags: [AI, LLM, Post-Training, Reinforcement Learning, Foundation Models]
---

今天最值得看的 paper 是 **Training Foundation Models with a Single Algorithm**。

Foundation model 的训练通常被切成 pretraining、midtraining、SFT 和 RL。每个阶段使用不同的数据、代码路径和优化方法，团队也会为它们维护不同的 trainer 与基础设施。这篇论文提出一个更统一的描述：这些阶段都可以写成对某个目标分布的 maximum-likelihood fitting，真正变化的是训练数据来自哪里、占多大比例，以及在什么时候出现。

这个视角把完整训练流程变成了 data mixture、schedule 和 compute allocation 的联合设计问题。

## 用 likelihood fitting 表达 RL

Pretraining、midtraining 和 SFT 本来就使用 token likelihood。论文基于 MaxRL，把 binary reward 下的 RL 更新写成对 success-conditioned distribution 的 likelihood fitting：模型生成多条 response，正确样本获得与 estimated pass rate 相关的权重，错误样本权重为零。

不同训练来源因此可以共享一种数据格式。每个样本包含 token sequence 和逐 token 权重：

- Pretraining 和 midtraining 对 corpus token 使用单位权重。
- SFT 屏蔽 prompt，只训练 demonstration token。
- RL 同样屏蔽 prompt，并根据 reward 与 sampled pass rate 给 response 分配权重。

Trainer 只需要优化一个 weighted negative log-likelihood。Static data 与 model generations 进入同一个 buffer，generation 和 optimization 可以异步运行，模型权重定期同步。

## Staged training 成为 data schedule 的一个特例

在 unified formulation 中，每个数据源都有随时间变化的 mixture weight。传统 staged pipeline 对应一种突变 schedule：某个阶段把全部权重交给一种数据，随后再整体切换。

Unified training 允许这些来源重叠。Pretraining corpus、midtraining data、demonstrations 和通过 verifier 的 model generations 可以同时出现在 batch 中，比例也可以在训练过程中增长、衰减或归零。

这带来一个实用变化：团队可以直接研究某份数据在整个 training trajectory 中的价值。评估重点也随之扩展到最终模型的 accuracy、coverage、retention 与后续学习能力。

## 统一训练改善了部分 compute-performance frontier

作者先用 124M 模型做受控实验，所有 run 共享 20B-token pretraining corpus 和 510M-token midtraining corpus，再通过可验证 reasoning puzzles 进行 RL。

表现最好的 unified configurations 在 Base、Extended 和更大 puzzle 的 generalization 测试上超过 staged baseline。按总训练 compute 比较时，staged training 在部分中等预算区间占优，unified training 在更小或更大的预算以及 generalization 场景中覆盖了更多最优点。

Retention 的差异也很明显。把 reasoning data 混入 pretraining 的 unified models 基本保持了 pretrain-only model 的 held-out loss 与通用 benchmark 表现；即使选择 retention 最好的 staged configuration，遗忘仍然更严重。

## Data mixture 暴露真实的能力权衡

论文还训练了一个 0.76B 模型，用 heterogeneous midtraining corpus 配合 Math12K RL。随着 midtraining data 比例增加，math accuracy 大体稳定，较高比例带来更好的 solution coverage 与 corpus retention；coding coverage 上升，coding accuracy 却可能下降。

调整 schedule 可以改变这种权衡。让 midtraining exposure 在训练中途归零，会进一步恢复 coding accuracy 并保留部分 SciQ 增益，同时牺牲一些 held-out corpus loss。单一 mixture ratio 没有普遍最优解，数据何时出现与出现多少同样重要。

更值得关注的是 continual learning。完成数学训练后，所有模型使用相同的 coding RL。此前采用 unified mixture 的模型最终在 HumanEval+ 和 MBPP+ 上获得更高 accuracy，其中 5% mixture 在四项主要指标上表现最好。训练期间保留多样数据，可能会让模型成为更好的下一阶段起点。

## 对 post-training infrastructure 的启发

这套方法很适合重新审视 SFT/RL 系统边界：

- 用统一的 token-weighted format 表达 corpus、demonstration、replay 和 verified generation。
- 让 rollout workers 与 trainer 异步运行，通过共享 buffer 解耦 generation throughput 与 optimization。
- 把 retention data 直接放进 mixture，用 schedule 控制遗忘与目标任务学习之间的平衡。
- 同时优化 data allocation 与 heterogeneous compute allocation，而非分别调整彼此隔离的训练阶段。
- 用最终 capability、coverage 和 continual learning 表现评估 midtraining 数据，而非只看该阶段结束时的 accuracy。

论文的证据仍有边界。主要实验规模较小，多数 run 只优化单个 RL task，也没有给出可预测的 data-mixture scaling law。它提供了一套统一的数学与系统接口，下一步需要在更大模型、多任务 GRPO 和真实生产 rollout 上检验。

## 今天另外两篇相关论文

**WorkFactory: Turning Professional Intents into Executable Work for Scalable Agent Training** 用 information support graph 描述专业任务所需的信息依赖，再通过 probe execution 发现并修复任务环境缺陷。它构建了覆盖 612 种职业的 112,686 个任务，WorkPilot-35B-A3B 在 GDPval-AA v2.1 上从 879 提升到 1565 Elo。

**Multi-Token Prediction Done Right** 发现标准 MTP gradient 包含一个低 signal-to-noise residual，会干扰 next-token 与 multi-token learning。Soft-MTP 通过 straight-through gradient routing 恢复有用的 KL self-distillation signal，只需一行代码修改，不增加参数或可测量开销，并同时改善 perplexity、speculative decoding 与下游 reasoning。

论文：<https://arxiv.org/abs/2610.40564>

另外两篇：<https://arxiv.org/abs/2610.37161>、<https://arxiv.org/abs/2610.38331>
