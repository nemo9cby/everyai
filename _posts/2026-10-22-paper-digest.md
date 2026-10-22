---
title: "Paper Digest: 2026-10-22"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Reinforcement Learning, Post-training, Systems]
---

今天最值得看的 paper 是 **Harness Scaling for Generalizable Agentic Reinforcement Learning**。

Agentic RL 通常固定一套 harness 训练：相同的 tools、system prompt、context policy 和 execution protocol。模型在这套环境里获得更高 reward，同时也可能记住运行时特有的 action patterns、信息布局和恢复策略。换一个 agent framework 后，训练收益未必保留。

Harness Scaling 把 harness diversity 明确纳入训练分布。作者先把 harness 拆成若干可组合因素，再用 balanced factorial coverage 选择一个紧凑的训练子集，让相同任务在受控的 runtime variation 下产生 rollouts。实验覆盖多个 benchmarks、model scales 和 RL algorithms，结果显示这种训练能够改善 unseen white-box harness 上的泛化，也能迁移到训练时没有见过的 external black-box harnesses。

## Harness 也是一种训练分布

一个 agent harness 会决定模型能够看到什么、如何调用工具、历史如何进入 context，以及失败后有哪些恢复路径。即使 base model、task 和 reward 完全相同，harness 也会改变 trajectory distribution。

固定 harness 的 RL 因此包含一种隐性捷径：policy 可以适应当前 runtime，而没有学会在不同 runtime 中稳定成立的 tool-use strategy。复杂 harness 也不天然解决这个问题。它可能提供更多功能，同时仍然只是一个训练点。

Harness Scaling 的关键是 structured variation。作者构造 factorized harness space，并选择覆盖不同因素组合的训练配置。相同 task 在多个配置中反复出现，policy 必须找到能够跨配置复用的行为规律。

## 为什么 balanced factorial coverage 重要

如果随机增加 harness variants，训练成本会快速膨胀，而且不同因素经常一起变化，难以判断泛化来自哪里。Factorial design 让有限数量的配置覆盖主要因素和组合关系，在 rollout budget 不变的情况下获得更有结构的环境多样性。

这也让 held-out evaluation 更严格。除了保留 unseen tasks，还可以保留 unseen harness combinations，检查模型是否只适应训练期间见过的工具和 prompt arrangement。

论文报告的结果跨越多种 model scales 和 RL algorithms。最值得注意的结论，是 structured white-box variation 能够迁移到 unseen external black-box harnesses，并且优于只通过一个复杂 black-box harness 训练的模型。训练时可控、可解释的 runtime variation，最终产生了更好的外部框架迁移。

## 对 agentic RL 基础设施的启发

第一，**每条 rollout 都需要完整的 harness metadata**。Policy checkpoint、tool schema、system prompt、context policy、execution limits 和 verifier version 应该进入同一条 lineage。缺少这些信息，很难分辨收益来自 weights、runtime，还是两者的配合。

第二，**评估矩阵要同时 hold out tasks 和 harnesses**。一个 checkpoint 在固定 runtime 上提高 pass rate，只能证明它适应了这套部署组合。跨 harness 的稳定增益，才更接近可迁移的 agent capability。

第三，**同一任务可以成为 harness 对照实验的基本单位**。在分布式 rollout 系统中，把同一个 prompt 分配到若干受控 harness variants，可以估计不同 runtime 因素的主效应和交互效应，同时减少 task difficulty 带来的噪声。

## 一个直接的 Ray / GRPO 实现思路

Rollout scheduler 可以先为每个 task 采样一组 balanced harness configurations，再把它们分配给 Ray workers。Worker 返回 trajectory reward、token cost、tool failures、context truncation 和 wall-clock latency。Aggregator 按 task 和 harness factor 分层统计，并在构造 GRPO groups 时控制环境差异。

真正困难的地方是 compute allocation。Harness coverage 太少，policy 容易继续过拟合；variants 太多，单个配置的数据不足，group comparison 也可能受到环境差异干扰。一个可行方向是先用小规模 factorial sweep 找出影响最大的 harness factors，再把 rollout budget 集中到这些维度，并持续保留少量 unseen combinations 作为泛化探针。

这篇论文给出的核心提醒很直接：训练 agent 时，环境配置本身就是 data。扩大 model、task 和 rollout 数量之外，harness diversity 也值得成为一条独立 scaling axis。

## 今天另外两篇相关论文

**PAUSE: Search-Free Steering of Coding Agents Before Their First Source-Code Edit** 在 coding agent 第一次修改源码之前插入一次 planner review。此时 agent 已完成大量 repository exploration 和 bug reproduction，又尚未因错误 patch 产生路径依赖。使用 Qwen3-Coder executor 和 GPT-5.6 Sol planner 时，PAUSE 在 OpenHands 的 SWE-bench Verified 上将 resolution 提高 16.8%，在 SWE-agent 的 Multilingual 上提高 14.7%。

**Cache Engineering for Agents** 让 agent 直接操作 active KV cache，通过 drop 和 repos 控制后续 attention，而无需改写文本轨迹或重新 prefill 保留的 context。其 rolling-drop policy 相比 threshold-triggered summarization 提高平均 benchmark score 1.7 points、降低成本 27.7%；配套 AKV serving engine 达到最高 1.2x output throughput 和 1.5x P95 latency speedup。

论文：<https://arxiv.org/abs/2610.29458>

另外两篇：<https://arxiv.org/abs/2610.29870>、<https://arxiv.org/abs/2610.28577>
