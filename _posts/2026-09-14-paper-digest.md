---
title: "Paper Digest: 2026-09-14"
categories: [Paper Digest]
tags: [AI, LLM, RLVR, GRPO, Post-Training, Code Agents]
---

今天最值得看的 paper，我会选 **DataFlex-RL: An Evaluation Platform for RLVR Data Policies**。

它得到的结论很朴素，也很容易让人不舒服：在一套受控的 GRPO recipe 里，uniform sampling 已经带来了很大的提升；给 rollouts 加上更聪明的 selection、reweighting 或 adaptive mixture，最终都没有产生可复现的额外收益。

Qwen2.5-7B-Base 经过 uniform GRPO 后，math、logic、science 三个领域的 domain-balanced average 提高了 7.76 分。作者随后比较 8 种 selection/reweighting 方法和 3 种 adaptive mixtures。每种配置都使用 12 个 matched seeds，所有 paired 95% confidence intervals 仍然包含 0。

这类 null result 对 post-training 团队很有价值。它提醒我们，一次小规模 sweep 里领先半分的方法，可能只是在 training noise、seed variance 和 evaluation choice 共同制造的浪花上短暂露头。

## Data policy 到底改了什么

论文先做了一个很重要的拆分。所谓 RLVR data policy，至少包含三类完全不同的 intervention：

1. Selection：rollout 和 reward 已经生成之后，决定哪些 responses 真正进入当前 update。
2. Reweighting：保留 responses，但改变每个 response 或 token 对 loss 的贡献。
3. Mixture adaptation：改变下一批 prompts 来自 math、logic、science 各 domain 的概率。

这三个动作经常被放在同一个“data optimization”标签下面，实际改变的对象不同，成本结构也不同。

Selection 发生在 rollout 之后，所以丢掉 response 并不会省下 generation cost。它还会减少真正参与 optimization 的 tokens。如果结果变好，原因可能来自 targeting signal，也可能只是 update volume 发生了变化。

Reweighting 保留所有样本，通过 continuous weights 调整学习强度。Mixture adaptation 则不碰当前 batch 的 GRPO loss，只改变未来看到的 prompt distribution。

DataFlex-RL 把 intervention 和 signal 分开记录。Solve rate、reward、advantage magnitude、token probability 都只是信号。同一个 advantage signal 可以被用来做 hard Top-K selection，也可以变成 continuous loss weight。这样才能回答一个更具体的问题：效果究竟来自信号，还是来自对训练过程采取的动作。

## 一套共同的 GRPO 实验底座

Baseline 每次从 math、logic、science 等比例采样 prompts，每个 prompt 生成 5 个 responses，经过 verifier 得到 rewards，再对所有有效 response tokens 应用标准 GRPO loss。

Primary experiment 使用 Qwen2.5-7B-Base，对 13 个 configurations 各跑 12 个 matched seeds，总共 156 runs。整套 release 包含 591 runs，涵盖 primary block、机制与强度 controls、Qwen base-model matrix，以及 model scale 和 family extensions。

Selection methods 包括：

- difffilter，用 group solve rate 保留难度处于中间区间的 prompt group
- maxvar，用 outcome reward variance 挑选更有区分度的 groups
- gfpo，使用 reward-per-token 信号
- topk，按 mean absolute advantage 做 hard selection

Reweighting methods 包括 Advantage Reweighting、prioritized-experience-style weighting、advantage softmax 和 difficulty-band weighting。Mixture methods 则根据 domain reward gap、UCB-style statistics 或 curriculum signal 动态调整三个领域的采样比例。

训练和 evaluation 使用共同的 driver、prompt template、verifier、rollout budget 与 optimization settings。最终 evaluation 覆盖 12 个 benchmarks，并先在 domain 内求平均，再给 math、logic、science 相同权重。

## 大增益来自 GRPO，细策略没有站稳

Uniform GRPO 的 Overall gain 是 7.76 points。这个结果说明模型确实有足够的 training headroom，实验并没有卡在一个完全学不动的 checkpoint 上。

加入 selection 和 reweighting 后，包含 baseline 的 9 个 policy means 只跨越 0.97 point，大约是 GRPO 自身增益的八分之一。

表现最好的 difffilter 相对 baseline 只有 +0.08，95% CI 为 [-0.67, 0.82]。其余方法的 point estimate 全部为负：

- maxvar：-0.58
- gfpo：-0.90
- topk：-0.38
- Advantage Reweighting：-0.28
- PER：-0.86
- softmax：-0.23
- diffband：-0.02

每一个区间都穿过 0。

更有意思的是 seed 数量改变了 selector 排名。原始 3-seed subset 中，topk 领先；扩展到 12 seeds 后，difffilter 的 mean 最高，topk 掉到 baseline 以下。如果研究停在前三个 seeds，故事会完全不同。

Adaptive mixture 也有相似结果。reward_gap、dump_ucb 和 tscl 相对 fixed equal mixture 的 point estimates 都是正数，分别为 +0.45、+0.61 和 +0.16，但三个置信区间仍然全部包含 0。Training logs 确认 domain proportions 确实发生了变化，说明策略成功改变了训练数据，只是这种改变没有稳定转化为 final accuracy。

## Evaluation 可以选出另一个赢家

论文最值得记住的第二个结果，是 evaluation-domain coverage 对结论的影响。

作者对同一批 9 个 Qwen2.5-7B-Instruct configurations 使用四种 summary：

- DB-12：12 个 benchmarks，先按 domain 平均，再让三个 domains 等权
- Macro-12：12 个 benchmarks 直接等权
- Item-12：按 evaluation-set size 加权
- Math-Heavy-6：5 个 math benchmarks 加 GPQA-Diamond，没有 logic benchmark

保留全部 12 个 benchmarks 的 Macro-12 和 Item-12，与 DB-12 的 rank correlation 都是 0.88。Math-Heavy-6 与 DB-12 的 Spearman correlation 却只有 -0.33。

Math-Heavy-6 会把 topk 和 diffband 排在前面，DB-12 则更偏向 gfpo 和 difffilter。原因很具体：某些 policy 在 logic 上强，在 science 上弱。一旦 evaluation 直接删掉 logic，它衡量的能力 profile 已经发生实质变化。

这比“多跑几个 benchmark”更严格。Benchmark suite 需要覆盖 training mixture 所声明的 domains，否则数据策略可能只是在未被测量的地方交换能力。Aggregation 的细节会影响分数，漏掉整个 domain 的影响更大。

## 对 post-training infra 的启发

DataFlex-RL 给训练平台提供了一套很实用的 experiment schema。

每次 data-policy run 至少应该记录：

- policy family：selection、reweighting 或 mixture adaptation
- driving signal：reward、solve rate、advantage、token probability 或 domain progress
- effective update tokens，而不只记录 rollout tokens
- 每个 domain 实际得到的 prompt proportion
- verifier 与 evaluation answer format 是否对齐
- 从 training 到 final evaluation 一直一致的 seed identity
- run-level scores、paired differences 和 confidence intervals

分布式 rollout 系统还需要保留足够的 provenance，让每个 response 的生成、验证、mask、weight 和 update contribution 可以重建。否则一个方法最终领先时，很难判断它找到了更有价值的数据，还是悄悄改变了 update volume 或有效 batch composition。

这篇论文也没有声称所有 data policies 都无效。它的结论有清楚边界：在这些 models、training lengths、hyperparameters 和 policy families 下，没有观察到可复现的额外提升。更长训练、不同模型、不同调参预算，仍然可能得到其他结果。

不过，举证责任已经更明确了。新的 RLVR data policy 需要击败经过充分训练的 uniform baseline，需要 matched seeds 和 paired uncertainty，也需要覆盖完整能力 domains 的 evaluation。单个 3-seed table 里高出零点几分，已经很难算充分证据。

## 另外两篇

第二篇是 **GraphAHA: Graph-Based Adaptive Search with Heterogeneous Actions for Test-Time Code Generation**。

它把 test-time code search 从 tree 改成 typed DAG。不同 trajectories 如果收敛到同一个 program，就合并为一个 code node，共享 downstream evaluation 和 search statistics。Hierarchical Thompson sampling 再动态决定是沿已有 successor 继续走，还是生成新 state，并在 sampling、reasoning、implementation、repair 之间分配预算。论文在 LiveCodeBench 和 CodeContests 的 20 个设置里拿下 18 个最佳，visible-test Pass@1 平均领先最强 baseline 4.1 points。

第三篇是 **Feyospace-v1: How the Cyber Mercury Seven Trained Frontier Cyber Models**。

一个 7 人团队构建了可重置的 coding、vulnerability、CTF、kernel-history、full-exploit、firmware 和 device-backed environments。候选 trajectories 经过 execution verification 与 evidence auditing 后，保留 164,269 条用于 long-context SFT。三个 checkpoints 在 CyberGym 上相对起始模型平均提升 23.76%。它最有价值的部分是完整 data engine：环境、teacher sampling、expert intervention、执行证据和训练数据形成一条可审计链路。

论文：<https://arxiv.org/abs/2609.06107>

另外两篇：<https://arxiv.org/abs/2609.12757>、<https://arxiv.org/abs/2609.08418>
