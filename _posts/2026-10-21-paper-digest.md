---
title: "Paper Digest: 2026-10-21"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Reinforcement Learning, Post-training, SWE]
---

今天最值得看的 paper 是 **Training the Whole Agent, Not Just the Model**。

它研究 coding-agent RL 里的一个核心耦合：policy 产生的 trajectories 受 harness 影响，新的 policy checkpoint 又会暴露新的 interaction failures。HarnessAdapt-RL 因此在 GRPO 训练期间持续调整三类组件：用于训练的 mechanism-level skills、可复用 tools，以及提供 runtime feedback 的 hooks。

在同一个 9B base model 上，HarnessAdapt-RL 相比 GRPO 把 SWE-bench Verified、Multilingual 和 Pro 的 pass@1 分别提高了 9.8、12.6 和 6.4 points，最终达到 57.6%、37.3% 和 39.7%。三套 benchmark 的平均成绩是 44.9%，GRPO 为 35.3%。

## 两类 failure，两种干预

论文把训练 trajectories 暴露的问题分成两类。

第一类是 task-specific 的 repair strategy failure，例如诊断方向错误，或者 patch 只修复表面症状。Frozen teacher 会比较同一任务的两条当前 policy trajectories，提取一条 mechanism-level skill。它解释某种 repair mechanism 在什么条件下成立、为什么有效，同时删去 repository-specific 的答案细节。

第二类是跨任务反复出现的 interaction failure，例如 editing command 失败、tool output 过长、提交前验证不足。另一个 teacher 会检查 tool calls、outputs 和 hook messages，提出新的 tools 或 runtime hooks，并直接实现候选 harness。

这两类干预的生命周期不同。Skills 只在训练阶段帮助 policy 学会 repair mechanism，评估时会全部移除。Tools 和 hooks 会保留在 deployment harness 中，因为它们改变了 agent 与环境交互的方式。

## Harness candidate 需要同时服务推理和学习

HarnessAdapt-RL 用两个指标筛选候选 harness：

1. **Pass rate**：rollouts 中成功解决任务的比例。
2. **Trainable rate**：rollout groups 中能够提供有效 policy update 的比例。

一个 proposal 只有在至少改善其中一项、同时不降低另一项时才会被接受。这个约束很有价值。更容易完成任务的 harness 可能让 reward group 全部成功，训练信号随之消失；过难或反馈混乱的 harness 又可能让 group 全部失败。Trainable rate 让 harness search 同时考虑当前表现和下一次 GRPO update 是否有足够的 reward variation。

每个 training epoch 最多执行五轮 tool/hook evolution，然后使用选中的 harness 收集新 rollouts、更新 policy。更新后的 policy 会产生新的失败分布，下一轮 harness proposal 再以这些 trajectories 为证据。

## Mechanism-level skills 比答案提示更容易留下能力

论文比较了三种 training guidance：near-answer hints、mechanism-level skills，以及跨任务维护的 general skill bank。

评估时移除全部 skills 后，mechanism-level 版本的平均 pass@1 是 38.3%，general skill bank 是 34.1%，near-answer hints 是 32.7%。Near-answer hints 在训练阶段能带来更高成功率，但移除提示后没有形成同等强的 repair ability。

Mechanism-level skills 保留了可迁移的 technical reasoning，又让 policy 自己完成 repository-specific diagnosis 和 implementation。这种信息粒度很适合 SFT/RL data design：teacher 提供机制，student 仍然必须经历完整的 action sequence。

## Policy 与 harness 要成对部署

这篇论文最有说服力的实验，是固定 HarnessAdapt-RL 训练后的 policy weights，只替换 evaluation harness。

使用 evolved harness H2 时，Verified、Multilingual、Pro 的 pass@1 分别是 57.6%、37.3%、39.7%。换回 base harness H0 后，成绩降到 52.8%、29.6%、35.0%，跌幅分别为 4.8、7.7 和 4.7 points。

同一套权重在不同 harness 下出现明显差距，说明训练得到的 artifact 包含 policy checkpoint、tools 与 runtime hooks。论文中的 skills 可以在 deployment 时移除，evolved tools/hooks 需要随 checkpoint 一起版本化。

## 对 code-agent RL 系统的三个启发

1. **Policy 和 harness 使用同一条 lineage。** 每个 rollout batch 都应记录 policy checkpoint、tool set、hook version、skill source 和 verifier outcome。
2. **Harness search 要跟随 policy 更新。** Pre-evolved harness 的平均成绩低于训练期间持续适配的版本，说明早期 trajectories 很快会过时。
3. **把 trainability 作为系统指标。** 除了 benchmark pass rate，还要追踪 zero-variance groups、有效 advantages 比例，以及每单位 rollout compute 产生的可训练样本。

这套流程很适合分布式 rollout infrastructure。Ray workers 可以共享 repository snapshots，并行评估多个 harness candidates；aggregator 计算 pass rate、trainable rate 和置信区间；accepted harness 再进入下一批 on-policy rollouts。真正困难的工程问题，是在有限 compute 下平衡 harness comparison 和 policy learning，并避免用少量随机结果接受退化 proposal。

## 今天另外两篇相关论文

**Self-Credit: Credit Assignment for Long-Horizon Agentic RL via Intra-group Rescaling** 让 policy 在 rollout 中引用产生关键证据的历史 turns，再用同一 prompt 下成功与失败 trajectories 的组内差异修正 credit，最后重加权 turn-level advantages。它在 BrowseComp-Plus、GAIA 和 xBench-DeepSearch 上都超过已有 agentic-RL baselines，为 sparse outcome reward 提供了一条轻量的 credit-assignment 路径。

**Evolving Agent Interfaces** 让 meta-agent 从 task trajectories 中构建和修改 computer-use interfaces，再通过 held-out validation 上的 population evolution 选择能够泛化的 interface repositories。Qwen3.8-27B 在 OSWorld V2 上达到 21.3% binary score，是同模型 fixed-interface baseline 的 3.8 倍；ScienceBoard success rate 从 41.4% 提高到 58.6%。

论文：<https://arxiv.org/abs/2610.28069>

另外两篇：<https://arxiv.org/abs/2610.28388>、<https://arxiv.org/abs/2610.28601>
