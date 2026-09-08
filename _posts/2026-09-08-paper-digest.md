---
title: "Paper Digest: 2026-09-08"
categories: [Paper Digest]
tags: [AI, LLM Inference, Discrete Diffusion, Speculative Decoding, Post-Training, Reinforcement Learning]
---

今天最值得看的 paper，我会选 **Unlocking Lossless Speedups in LLMs via Discrete Diffusion**。

作者提出了一类 diffusion-augmented LLM，并将模型命名为 Uno。它保留标准 autoregressive LLM 的训练和输出分布，同时增加一组轻量 diffusion weights，一次并行提出多个 token。随后，原始 AR weights 验证这些候选 token，只接受通过 rejection sampling 的最长前缀。

这带来一个很干净的职责划分：

- AR weights 决定模型质量，继续使用 next-token prediction、SFT 和 RL 训练。
- Diffusion weights 负责并行 drafting，只优化生成速度。
- Ψ-Spec sampler 负责验证和修正，确保最终样本来自原始 AR 分布。

Uno 在所有测试 batch size 上都比 base AR 更快。基于 Qwen3-8B 的版本在最大 batch 下获得 1.6 倍 system-throughput speedup，在 batch size 1 下获得 2.5 倍 per-request speedup。更值得 post-training 团队注意的是，它把 math 和 code expert 的端到端 RL 训练最高加速了 40%。

## 一套模型里放两条 pathway

Uno 给每个 AR weight matrix 配置一组 LoRA diffusion adapter。Drafting 时使用 `θAR + θΔ`，verification 时只使用 `θAR`。

这种设计让两条 pathway 共享大部分参数和 KV cache，额外显存小于维护独立 draft model。对于 Qwen3-8B，新增 diffusion adapters 有 0.35B 参数。相比之下，论文中的 EAGLE-3 drafter 有 0.40B 参数，DFlash drafter 有 1.05B 参数。

训练 diffusion weights 时，base AR weights 完全冻结。模型学习从一个被随机 token 污染的 block 中恢复出 AR teacher 会逐 token 生成的内容。作者将这个阶段称为 **Diffusion Distillation**。

## 为什么训练目标直接优化 acceptance

一个 diffusion draft 能带来多少加速，取决于 AR verifier 连续接受多少 token。Uno 的 loss 由两部分组成：

1. `LDCD` 让 diffusion pathway 匹配 frozen AR teacher 的 token distribution。
2. `LTV` 最小化两条分布之间的 total variation distance，直接提高连续 token 被接受的概率。

Ablation 里，单独使用 `LTV` 的效果最好。论文自己的 8B 模型仍保留一个很小的 `LDCD` 权重，设置为 `α=0.01, β=1`。Qwen3-8B 版本使用 `α=0, β=1`。

训练采用 block-size curriculum。Qwen 版本按 `2, 4, 6, 8, 12, 16` 逐步增加 block size，共训练 14.7B tokens，耗时约 32 小时，使用 32 张 H200。自研 8B 模型的 adapters 训练 7B tokens，耗时约 60 小时，使用 64 张 H200。

## Ψ-Spec 如何保证 lossless

每轮生成包含两次 forward pass：

1. Diffusion pathway 一次提出一个 token block。
2. AR pathway 并行计算这些候选 token 的概率，并执行标准 speculative-decoding rejection correction。

第一个 token 直接来自 AR weights，因此一定会被接受。后续 token 按顺序验证，遇到拒绝时由 verifier 从修正后的 residual distribution 采样替代 token。全部通过时，verifier 还会额外产生下一个 token。

这个过程严格保留 base AR model 的 target distribution。Diffusion adapters 可以改变 acceptance length 和速度，无法改变最终输出质量。论文用“lossless”表达的正是这层分布保证。

Ψ-Spec 提供两种配置：

- Linear sampler 只采样一条 candidate sequence，适合高 batch、compute-bound 的 system throughput 场景。
- Tree sampler 采样多条候选并用 tree attention 并行验证，利用 batch size 1 时尚未吃满的计算资源，优化单请求速度。

同一模型可以根据 serving concurrency 切换 sampler。论文发现，Linear sampler 的 block size 继续增大后，acceptance 提升不足以覆盖计算开销，`B=4` 的 system throughput 最高。

## 真实 serving 需要看最大 batch

许多 parallel decoding 结果集中在 batch size 1。Agent workload 经常同时存在多个请求、分支、tool call 和 retry，单用户也可能形成较高 concurrency。此时 serving engine 会批量处理请求，模型逐渐从 memory-bound 进入 compute-bound，验证更多候选的成本会明显上升。

论文同时报告 batch size 1 和单张 H200 能容纳的最大 batch。Qwen3-8B 版本在最大 batch 下超过 5700 tokens/s，比 base AR 快 1.6 倍；batch size 1 的吞吐达到 base AR 的 2.5 倍。Uno 在所有 batch size 上都优于 EAGLE-3 和 DFlash，并共享 draft 与 verifier 的 KV cache。

自研 8B Uno 在多个 agentic benchmark 上也给出了很强的模型结果：

- SWE-bench Verified：68.4
- Terminal-Bench v2.1：39.6
- τ2 Telecom：90.1
- AA-LCR：68.0

它在 agentic tool use、coding 和 long-context reasoning 的全部对比项上超过 DiffusionGemma-26B-A4B 与 Mercury 2。最大 system throughput 为 5255 tokens/s。

## 对 RL post-training 的意义

作者在 SFT checkpoint 后训练 diffusion adapters，随后用 DAPO 分别训练 math、code、tool-use 和 web-search experts。RL 阶段只更新 AR weights，diffusion adapters 保持冻结，并继续用于 rollout generation。

即使 policy 已经发生变化，adapters 的 tokens-per-forward-pass 只下降约 6%。Math 和 code experts 的端到端训练时间最高缩短 40%。Tool-use 与 search expert 的收益较小，因为外部工具调用占据了更多 wall time。

这给分布式 RL 系统一个很实用的判断标准：先测 rollout generation 在总训练时间中的比例。Generation-bound workload 最值得引入 Uno；tool-latency-bound workload 的收益会被环境等待时间稀释。

下一步值得测量的是不同 context length、response length、concurrency 和 policy drift 下的 rollout tokens per GPU-second，以及 adapter refresh 的最佳频率。Uno 已经证明，pre-RL adapter 可以跨越一段 policy update 继续工作。更长训练周期里的 acceptance decay 仍值得单独研究。

## 另外两篇

第二篇是 **FlowBalance: Verifier-Grounded Self-Improvement from On-Policy Reasoning Experience**。

FlowBalance 用 verifier outcome 校准 privileged same-model guidance。Positive-advantage trajectory 保留 guidance，negative-advantage trajectory 反转 guidance，没有 outcome preference 的 group 关闭 guidance。它在 Qwen3-4B 与 Qwen3-8B 数学 reasoning 上超过 FlowRL，同时改善训练稳定性、response-length collapse 和正确策略多样性。

第三篇是 **Verify Before You Distill: Prompt-Level Teacher Gating for On-Policy Distillation**。

TGOPD 先用 verifier-scored teacher probes 判断每个 prompt 上的 teacher reliability。通过 gate 的 prompt 使用 dense OPD，其余 prompt 使用 verifier-grounded GRPO。它在数学、code 和 instruction following 的六个 single-domain setting 中全部超过 vanilla OPD，并把一组 4B 实验里的 teacher-node GPU utilization 从 9.8% 提高到 78.9%。

论文：<https://arxiv.org/abs/2609.04010>

另外两篇：<https://arxiv.org/abs/2609.03241>、<https://arxiv.org/abs/2609.02998>
