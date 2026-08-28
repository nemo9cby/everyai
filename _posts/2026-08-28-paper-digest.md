---
title: "Paper Digest: 2026-08-28"
categories: [Paper Digest]
tags: [AI, Post-Training, Test-Time Training, Reinforcement Learning, Self-Distillation]
---

今天最值得看的 paper，我会选 **TTPO: Test-Time Policy Optimization**。

它研究一个很棘手的问题：没有 ground-truth label，模型还能不能在 test time 继续训练自己？

最直接的办法是对同一道题采样很多条 rollout，用多数票当 pseudo-label。可是在 competition math 这种高难度数据上，多数票经常也会答错。论文在 Qwen3-1.7B 的 AIME 2026 test-time training 中观察到，约 85% 的 prompt 都拿不到正确的多数答案。把这样的伪标签直接当 reward，或者塞给 distillation teacher，错误会被整个训练过程放大。

TTPO 找到了一条仍然可靠的信号：多数票虽然可能错，与多数票不一致的 rollout 通常也确实是错的。

在论文的统计中，即使 pseudo-label 错了，disagreeing rollouts 里仍有约 79% 没有得到 ground-truth answer。你可能不知道正确答案是什么，却常常可以安全地判断某条轨迹不值得强化。

## 同一组 rollout，走两条训练分支

TTPO 对每道题采样 64 条 rollout，对 final answer 做数学等价性聚类，再选最大 cluster 作为 pseudo-label。轨迹随后被分成两组：

1. 与 pseudo-label 一致的 positive samples
2. 与 pseudo-label 不一致的 negative samples

Positive samples 进入 On-Policy Self-Distillation。Teacher 和 student 使用同一个 model，teacher 额外看到 pseudo-label，并以 thinking mode 对 student 自己的 completion 做逐 token 重打分。

即使 pseudo-label 本身答错了，这条分支仍有一个保护条件：positive sample 产出的答案与 pseudo-label 相同。Teacher 接收到的是 student 自己已经给出的答案，因此更新更接近把 thinking-mode reasoning 蒸馏给当前 policy，错误答案不会凭空把 trajectory 拉向另一个方向。

Negative samples 进入 grouped RL。它们收到负 advantage，训练只使用“这条轨迹没有进入多数 cluster”这一事实，不需要知道多数答案的具体内容。

这种 asymmetric objective 很精巧。Dense distillation 被放在内部一致的样本上，coarser RL penalty 被放在 disagreement 仍具信息量的样本上。两种训练信号共用一次 rollout generation，承担的风险却被分开了。

## Token selection 再缩小错误半径

TTPO 还没有把 sequence-level 标签平均广播给所有 token。

在 distillation 分支，它根据 teacher-student divergence 与 student entropy 给 token 加权。已经收敛、teacher 和 student 几乎一致的位置权重很低，梯度集中在仍有学习价值的位置。

在 RL 分支，它只处罚 top 50% 的高分 token。筛选目标是 confident errors，也就是 model 以较高置信度生成、却在当前局部上下文中表现异常的 token。失败轨迹里可能包含很多正确推导，如果每个 token 都承受同样的负梯度，局部正确能力会一起受损。

Ablation 很清楚。去掉 positive token weighting，三个 benchmark 都退步；去掉 negative token masking，HMMT 2026 从 31.6 降到 29.5，BRUMO 2025 从 54.7 降到 50.0。TTPO 的提升依赖 routing，也依赖两条分支各自的 token-level credit assignment。

## 无标签训练追平有标签 baseline

论文在 Qwen3-1.7B、4B 和 8B 上测试五个 competition math benchmark。

在 OpenThoughts setting 中，GRPO 和 OPSD 都能看到 ground-truth label，TTPO 只能看到多数票。三种规模上的平均结果分别是：

- 1.7B：TTPO 40.1，OPSD 39.7
- 4B：TTPO 58.6，OPSD 58.4
- 8B：TTPO 62.6，OPSD 61.7

在纯 test-time training setting 中，所有方法都没有 label。TTPO 把 Qwen3-1.7B 的三项平均分从 38.0 提高到 45.2，比 TTRL 高 5.0 分，比 OPSD-TTT 高 3.3 分。

Qwen3-4B 经过 TTPO 后达到 61.1，超过未经训练的 Qwen3-8B 的 60.7。8B 本身也从 60.7 提升到 65.3。

训练数据只来自某一个 benchmark 时，另外两个 benchmark 的结果也会提高。这个 cross-benchmark transfer 说明模型学到的能力没有局限在当前题集。

还有一个有趣现象。使用完美 ground truth 的 TTPO，结果反而低于使用 pseudo-label 的版本。难题的正确答案很难被当前模型采样出来，ground truth 会导致许多 problem 没有 positive sample，distillation 分支随即失去梯度。Majority answer 更贴近 model 当前能力，能够维持 positive-negative balance。随着 policy 变强，多数票质量也会提高，下一轮训练信号跟着改善。

## 对 rollout infrastructure 的启发

TTPO 把一批不同计算形态塞进同一条 post-training pipeline：

- 每道题采样 64 条、最长 16K token 的 rollout
- 对 final answer 做 extraction 和 equivalence clustering
- 从 positive 与 negative 各选 4 条用于 update
- 用 thinking-mode teacher 做逐 token log-probability
- 用 grouped RL 计算 negative advantage
- 为两条分支构造不同 token mask

这里最大的系统机会是复用。64 条 rollout 已经承担了投票、训练和评估三种角色，generation cache、answer cluster、teacher score 与 policy version 必须绑定在一起。Pseudo-label 会随 checkpoint 改变，所以每次 update 也要保留形成多数票的 rollout group，避免异步 worker 把不同 policy 版本的数据混进同一个 group。

它的成本也不能略过。每道题 64 次 long-context generation 很贵，论文训练时只从中挑 8 条参与 gradient update。生产系统需要关注 vote confidence、自适应 rollout 数量、early stopping，以及相同 answer cluster 的增量统计。简单题很可能在远少于 64 条时就形成稳定多数，难题再使用完整预算。

TTPO 最有价值的地方，是它对 noisy supervision 做了细粒度拆解。一个 pseudo-label 的内容可能不可信，agreement structure 仍然包含信息。把不同强度的信号交给适合它们的 optimization branch，就能在没有外部答案的情况下，让 model 从自己的 rollout distribution 里持续提取训练信号。

## 另外两篇

第二篇是 **What Makes Good Agentic Data? An ACE Lens on Data Generation for LLM Agents**。

它把 agentic data 表示成四元组：environment、task、interaction trajectory 和 verifier，再用 Accuracy、learner-relative Complexity、divErsity 三个维度讨论数据生成。对 agent post-training 来说，这个框架很实用。一条 trajectory 通过 verifier，只能说明它有效；它对当前 model 是否太容易、是否覆盖新的 failure mode、是否与已有数据高度重复，还需要独立判断。

第三篇是 **Training Agents to Evolve with Their Harness: TaoLive Digital Avatar Agent Technical Report**。

它提出 Harness-Aware Training，对 skill identifier、tool schema、prompt structure 和 hook function 做 task-preserving augmentation，再依次使用 HSA-SFT、general on-policy distillation 和 HSA-RL。目标是让 compact model 适应不断变化的 harness。部署结果也很完整：单张 NVIDIA H20 上 P50 latency 3.4 秒、P95 8.1 秒，并在淘宝直播的线上 A/B test 中带来正向 GMV 与 item-page view 变化。

论文：<https://arxiv.org/abs/2608.27448>

另外两篇：<https://arxiv.org/abs/2608.27260>、<https://arxiv.org/abs/2608.15763>
