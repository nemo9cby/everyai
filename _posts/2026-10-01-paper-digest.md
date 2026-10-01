---
title: "Paper Digest: 2026-10-01"
categories: [Paper Digest]
tags: [AI, LLM, Post-training, Reinforcement Learning, On-policy Distillation, Agents]
---

今天最值得看的 paper 是 **The Teacher Is a Direction, Not a Destination: Extrapolating RL-Induced Representation Residuals in On-Policy Distillation**。

On-policy distillation 通常让 student 在自己的 trajectories 上匹配 teacher 的 next-token distribution。更激进的方法会进一步外推 teacher 隐含的 reward，希望 student 超过 teacher。

RIDE 提醒我们，真正有用的 RL change 未必能在 logits 中被完整观察。Language-model head 会对 hidden-state change 做各向异性的压缩，而 sampled-token log-probability ratio 自带噪声。一旦在 output space 继续放大这类信号，训练容易变得不稳定。

论文选择直接监督 representation：比较 RL teacher 与其 pre-RL base checkpoint，在每一层、每一个 token 上计算 hidden-state residual，再把 student 的训练目标放到 teacher 之外，沿着同一 residual 继续前进。

## Teacher 提供的是一条方向

设 base checkpoint 的某层表示为 `h_base`，RL teacher 的对应表示为 `h_rl`，那么 RL-induced residual 可以写成：

`r = h_rl - h_base`

普通 feature matching 会让 student 靠近 `h_rl`。RIDE 的 target 则继续沿 `r` 外推：

`h_target = h_rl + αr`

这里的 α 控制 student 越过 teacher 的幅度。论文证明，在给定 sampled trajectory 的条件下，这个 regression objective 等价于最大化一个由 residual 定义的线性 directional reward，同时用以 teacher 为中心的 quadratic penalty 约束偏移。

这个视角很有吸引力。RL checkpoint 与 base checkpoint 共同定义了 policy improvement 的方向，student 学习的对象也因此包含“RL 改变了什么”。

## 为什么 output-space extrapolation 会失真

Logit-space 方法依赖 teacher 与 reference 在 sampled tokens 上的 log-probability ratio。这里有两个问题。

第一，LM head 对不同 hidden directions 的传递强度差异很大。一些在 representation 中明显的 RL change，到 logits 时只剩很小的权重。

第二，student 自己采样的 token 带来较大的估计噪声。Extrapolation 会同步放大 signal 和 noise。Teacher 如果离 base 很近，这个问题更加明显，论文观察到 output-space extrapolation 在这类设置中反而会让 student 退化。

RIDE 绕开了最后一层投影，直接使用 teacher-base representation residual。监督信号更密集，也更明确地绑定到 RL 产生的内部变化。

## 实验信号

论文评估了四组 base/RL-teacher pairs，覆盖不同模型规模、架构和 pretraining lineages。RIDE 在每一组中都接近或超过 RL teacher，并且是所有比较方法中唯一在平均结果上超过 teacher 的方法。

更重要的结果来自稳定性：当 teacher 与 base 的距离较小时，output-space extrapolation 经常降低 student 表现，RIDE 仍然保持一致优势。

这说明 checkpoint pair 本身可以被当作训练信号。一个 RL model 除了提供答案分布，也携带相对 base 的内部 update geometry。

## 落地时真正昂贵的部分

RIDE 的代价同样清楚。每层、每 token 的 hidden-state supervision 会增加 activation memory、teacher forward 的存储压力，以及多卡场景中的通信量。实际系统很可能需要回答几个工程问题：

- 只监督 selected layers 能保留多少收益
- residual 能否低秩化、量化或在线压缩
- teacher、base 与 student 分布在不同设备时，怎样减少 activation transfer
- checkpoint distance 多小时，α 应该怎样设定
- 架构不完全一致时，representation alignment 如何完成

对 code 或 reasoning post-training，一个直接的实验是固定 rollouts 与算力预算，比较 KL distillation、token-level Direct-OPD、普通 hidden-state matching 和 RIDE。除了 benchmark accuracy，还要记录 peak memory、tokens per second、跨设备带宽和 residual norm 的 layer-wise 分布。

真正值得验证的问题是：**RL 训练得到的能力，能否被压缩成一组便宜、稳定、可迁移的 representation directions？**

## 今天另外两篇相关论文

**False Frontiers** 研究 self-evolving search agents 中的 co-cheating。Proposer 与 solver 会逐渐共享同一种错误，内部 agreement 持续上升，外部 correctness 却停滞甚至下降。CrossFit 用互不重叠的 source partitions 打断 feedback ancestry，在 4B 和 9B 模型上把 false-agreement mass 分别降到 3.0% 和 3.7%，并显著提升七个 downstream search benchmarks。

**Mid-Harness** 把 test-time compute 放在 terminal agent 与执行环境的边界。系统先生成多个候选 action，再由 verifier 选择一个真正执行。在 TerminalBench-Lite 上，强 verifier 配合八个候选，把 TMAX-9B 的 Pass@1 从 50.00% 提升到 68.03%。它提示我们，危险 action 真正改变环境以前，多花一次选择成本可能比重新采样整条 trajectory 更划算。

论文：<https://arxiv.org/abs/2609.36484>

另外两篇：<https://arxiv.org/abs/2609.39102>、<https://arxiv.org/abs/2609.39982>
