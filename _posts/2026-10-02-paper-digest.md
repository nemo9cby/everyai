---
title: "Paper Digest: 2026-10-02"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Reinforcement Learning, RLVR, Post-training]
---

今天最值得看的 paper 是 **Cross-Benchmark Transfer from RL on Agentic Coding Tasks**。

它问了一个 code-agent post-training 里很关键的问题：在一小批高质量 coding tasks 上做 RL，学到的能力能不能跨 repository、benchmark、agent harness 和任务时长继续生效？

作者选择 Kimi K2.7 Code，一个 1T parameters、32B active parameters 的 MoE 模型。训练数据只有 1,700 个 expert-built tasks，其中 1,000 个是 repository tasks，700 个是 terminal tasks。训练也很克制，只做一轮 GSPO，并用 rank-32 LoRA 更新模型。

结果覆盖六个外部 benchmark，而且每一个都提升了：

- SWE-Bench Pro：60.1% → 64.8%
- DeepSWE：31.0% → 43.4%
- Terminal-Bench 2.1：67.4% → 82.0%
- Terminal-Bench 3：1.4% → 12.1%
- Terminal-Bench 4：0.0% → 7.6%
- SWE-Marathon：5.0% → 25.0%

更有分量的信号来自 transfer。部分 benchmark 在训练数据收集完成以后才发布，两个 evaluation harness 也从未出现在训练中，提升依然成立。

## Reward 在教模型完成“最后一公里”

这篇论文最值得抄的部分，是 reward design。

Repository tasks 同时包含两类 hidden tests：

- fail-to-pass tests：检查新需求是否真的实现
- pass-to-pass tests：检查原有行为是否被破坏

Reward 会按照完成的 target checks 比例提供 partial credit。只要任何 pass-to-pass test 失败，整条 rollout 的 reward 直接归零。

可以把它写成一个很直观的规则：

`reward = target checks passed / all target checks`

同时加上 hard gate：

`if any regression: reward = 0`

Partial credit 让多个不完整方案之间仍然存在可学习的梯度。Regression gate 则明确告诉模型，新增功能不能拿已有行为做交换。

Terminal tasks 没有 pass-to-pass tests，reward 来自 expert-written hidden verifier 中满足的 criteria 比例。两类任务共同强调可执行、细粒度、与需求一一对应的反馈。

## 为什么这些行为能够跨 benchmark 泛化

作者分析了 base model 的失败轨迹。DeepSWE 中，大多数失败已经非常接近成功：失败 runs 通过 target tests 的中位数达到 86%，其中 84% 仍然保住了所有 pass-to-pass tests。

这说明模型缺少的常常是最后几步工程纪律。Paired trajectories 里，RL 后的模型更少出现四类问题：

- 漏掉需求中的某一项
- 只测试当前实现已经覆盖的情况
- 改坏本来需要保持不变的行为
- 根据未经验证的假设完成实现

这些模式并不依赖某一个 repository。它们是 coding agent 在多数真实任务里都会遇到的通用失败，因此在 expert-built tasks 上形成的策略可以迁移到新的 benchmark 和 harness。

效率也同步改善。RL 后的模型在 DeepSWE 和 Terminal-Bench 3 上，median agent steps 分别减少约 35% 和 24%。模型解决了更多任务，同时走了更短的轨迹。

## 对 code RL 的实际启发

训练任务数量并不大，真正昂贵的部分在 task construction 和 verifier quality。每一条 requirement 都需要可执行检查，已有行为需要 regression tests 保护，reward 还要保留足够细的 partial credit。

一个值得复现的最小实验是：

1. 固定一批 repository tasks 和 rollouts
2. 比较 binary success reward 与 requirement-level partial credit
3. 再加入 pass-to-pass regression gate
4. 在完全不同的 repositories 和第二套 harness 上评估
5. 同时记录 Pass@1、agent steps、rollout tokens 和 verifier cost

论文里的结果说明，code-agent RL 的核心资产很可能是可验证任务的设计质量。1,700 个精心构造的任务，已经足以让一个大模型在多个外部环境中表现出更可靠的工程行为。

## 今天另外两篇相关论文

**Sharpening Tax in Post-Training** 发现，post-training 普遍提升 Pass@1，却可能降低 repeated sampling 下的 solution coverage。模型在同一道题上的多次尝试变得更加一致，额外 test-time compute 因此更难发现新的解法。论文提出 Sharpening Tax 衡量这种 test-time scalability 损失，并用 difficulty-adaptive sampling 同时改善单次准确率和 Pass@K。

**On-Policy or Off-Policy Learning?** 把 rollout policy、token-level KL direction 和 learning rate 分开控制。实验显示 forward KL 对 rollout source 相当稳健，reverse KL 对 student-generated rollouts 更敏感；forgetting 和 update sparsity 则更多由 learning rate 决定。这对在线 rollout 成本很高的 distillation pipeline 很有现实意义。

论文：<https://arxiv.org/abs/2610.00890>

另外两篇：<https://arxiv.org/abs/2610.01509>、<https://arxiv.org/abs/2609.35259>
