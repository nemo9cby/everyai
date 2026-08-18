---
title: "Paper Digest: 2026-08-18"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, Post-Training, Code Agents, Agent Harness]
---

今天最值得看的 paper，我会选 **ClawGym II: Exploring Black-Box RL on Agent Harness**。

训练一个能在真实环境中工作的 agent，最麻烦的部分往往发生在 model 之外。OpenClaw、Claude Code 这类 harness 会管理工具、状态、重试、上下文压缩、并发任务和长时间执行。对用户来说，这些能力组成了完整产品。对 RL pipeline 来说，它们却像一个黑箱：一次任务可能触发多轮模型调用，轨迹发生分叉，环境持续变化，传统的单轮 prompt-response rollout 很难描述它。

ClawGym II 的核心贡献，是把 RL 的观察点放在 model serving boundary。Harness 可以继续按照原来的方式运行，训练系统通过 serving proxy 捕获模型调用，再从这些调用中恢复可优化的多轮轨迹。

## 把真实 harness 变成 rollout environment

论文搭建了一套 sandbox-based execution infrastructure。每个任务环境和 harness 都运行在临时隔离的 sandbox 中，系统可以并发启动大量长轨迹，同时控制环境污染和任务之间的状态泄漏。

真正关键的一步发生在模型调用层。Serving proxy 位于 harness 和 policy model 之间，记录每次请求与响应。这样，训练系统无需知道 harness 内部怎样规划、怎样选择工具、何时重试，也不要求 OpenClaw 或 Claude Code 改造成标准 RL environment。

这个边界选择很聪明。复杂 agent system 的实现变化很快，训练接口如果紧贴某个 harness，数据和算法会被它的内部结构锁住。Model API 是相对稳定的公共边界，捕获它，就能让多种 agent runtime 共享同一套 policy optimization infrastructure。

## 为什么要恢复 prefix tree

长任务中的模型调用并不总是一条简单链。Harness 可能重试某一步、并行探索多个方案，或者从同一段历史产生不同 continuation。把所有调用强行摊平成独立样本，会丢掉共享 prefix，也会破坏 action 与后续结果之间的关系。

ClawGym II 把捕获到的调用组织成 prefix tree。共享历史只保留一次，不同 continuation 成为树上的分支。作者进一步调整 critic-based PPO 和 critic-free GRPO，让它们能够在恢复出的 tree structure 上优化。

这件事对 credit assignment 很重要。一次成功可能来自后半段的修复，一次失败也可能发生在多个正确步骤之后。树结构至少保留了分支从哪里开始、哪些状态被多个 rollout 共享，以及 reward 应该沿哪条路径回传。

## Black-box RL 的实验结果

论文使用 Qwen3-30B-A3B，通过不同 harness 进行训练。在 ClawGym-Bench 上：

- 通过 OpenClaw 训练，Pass@1 提升 9.98 points
- 通过 Claude Code 训练，Pass@1 提升 14.81 points
- 训练在 200 到 400 个 optimization steps 内保持稳定
- 在更困难的 JobBench 和 OfficeQA 上也得到一致提升

这些结果说明，复杂 harness 产生的真实交互轨迹可以成为有效的 RL 数据。模型学到的能力也没有完全局限在单一 benchmark。

## Mix-harness training 更值得关注

论文还支持 mix-harness training，让一个 policy 同时通过异构 harness 接受优化。

这接近真实 deployment。相同 backbone 可能被放进 coding agent、browser agent、office agent 和内部 workflow。每个 harness 有不同的工具协议、上下文组织、失败恢复和调用频率。如果只能为每个 runtime 单独训练一份 policy，维护成本会迅速膨胀，也很难积累跨环境的通用能力。

Mix-harness training 提供了一个统一方向，但也带来几个值得继续研究的问题：

- 不同 harness 的 trajectory length 和 reward density 差异很大，怎样避免某一类 rollout 主导梯度
- Harness-specific action distribution 会不会互相干扰
- 长轨迹异步生成时，stale policy 对 PPO 或 GRPO 的影响有多大
- 同一个任务在不同 harness 中成功，模型究竟学到了通用策略，还是适应了 runtime 的固定习惯
- 在训练中完全没见过的新 harness 上，能力能否迁移

这些问题决定了 mix-harness training 最终会形成 general agent policy，还是一组共享参数的局部策略。

## 对 agent post-training 的启发

ClawGym II 给出了一条很具体的工程路线：保留成熟的 agent runtime，在 serving layer 统一采集轨迹，再用 tree-aware optimization 训练 policy。

这会改变数据建设方式。团队无需先把生产 harness 重写为 gym-style API，也可以把真实任务执行接入 post-training。OpenClaw、Claude Code 和 domain-specific agents 可以成为不同的 rollout generators，统一进入 trajectory store 和 advantage pipeline。

接下来最值得做的实验包括：

- per-harness advantage normalization 与 balanced sampling
- 针对长短轨迹的 credit assignment ablation
- 异步 rollout 下 policy lag 的稳定性边界
- 同任务跨 harness 的 transfer matrix
- 训练时混合多个 harness，测试一个 held-out harness
- 将 tool outcome、state transition 和 final success 分解为不同 reward channels

最后一项还可以和今天第三篇 SA-MRPO 连接起来。当格式、工具调用和最终正确性具有不同饱和速度时，固定 reward weights 很容易浪费 gradient budget。Harness-level diversity 与 reward-level diversity，需要在同一套训练系统中共同处理。

## 为什么今天选它

Agent 能力来自 model、harness、environment 和 training loop 的耦合。ClawGym II 给出了一套能落地的连接方式：用 sandbox 扩展真实执行，用 serving proxy 穿透黑箱，用 prefix tree 保存多轮分支，再让 PPO 和 GRPO 直接优化这些轨迹。

对正在做 code agents、computer-use agents 和 heterogeneous post-training infrastructure 的团队，这篇论文比一套只对单一 benchmark 生效的 agent recipe 更值得细读。

## 另外两篇

第二篇是 **StateM: Reaching 95.3% Raw Accuracy, or a $15 Frontier Run, on Terminal-Bench 2.1 via Harness Scaling**。

StateM 不修改 model weights，而是在 runtime 中加入 durable states、phase-local context、checked transitions、recoverable runbooks 和 versioned procedural practices。它把 GPT-5.5 xhigh 在 Terminal-Bench 2.1 上提高到 92.1%，并用 GPT-5.6 Sol xhigh 达到 95.3% raw accuracy。更有意思的是，同一套 runtime 和 runbook structure 也能提升更弱、更便宜的模型。它提醒我们，模型之外仍有大量可审计、可复用的 reliability gain。

第三篇是 **Learn What's Left, Not What's Mastered: Saturation Aware Advantage Reweighting for Multi-Reward Policy Optimization**。

SA-MRPO 研究 multi-reward GRPO 中的固定 scalarization 问题。它分别标准化每个 reward objective，再根据 batch-level saturation estimate 降低已经学会的目标权重，把更新集中到仍有提升空间的 objective。方法在数学 reasoning、adaptive reasoning 和 coding benchmarks 上提高更难的 correctness objective，同时维持已经饱和的简单目标。对于同时使用 correctness、format、tool-use 和 safety verifiers 的 post-training pipeline，这是一项很容易实现和验证的改动。

论文：<https://arxiv.org/abs/2608.16798>

另外两篇：<https://arxiv.org/abs/2608.15089>、<https://arxiv.org/abs/2608.16072>
