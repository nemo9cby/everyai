---
title: "Paper Digest: 2026-07-29"
categories: [Paper Digest]
tags: [AI, Code Optimization, Reinforcement Learning, GRPO, Code Agents, Post-Training]
---

今天最值得看的 paper，我会选 **Reinforcement Learning for Code Optimization**。

Code RL 已经有一条相对清晰的路径：模型生成程序，sandbox 执行 hidden tests，通过就得到 reward。

当目标加入 execution time，训练问题立刻复杂很多。Runtime 会受到 test size、机器负载、进程调度、缓存与测量噪声影响。大量 rollout 连正确代码都写不出来，能够比较快慢的样本更少。Reward 若直接混合 correctness 与 speed，模型还可能拿到 fast-but-wrong 的部分分数。

这篇 125 页的论文把这些问题拆成三个层次：

1. 怎样构造能测出优化差异的 code problems。
2. 怎样把 correctness 与 speed 组合成可学习的 reward。
3. 怎样让 async GRPO 在 sparse、noisy timing signal 下保持稳定。

它最有价值的地方，是把 benchmark、execution service、reward 与 distributed RL loop 放进同一个实验框架。

## DMC-Optim 先让速度变得可测

作者从 12,275 个 raw DMC problems 出发，重新执行 human solutions、补充 tests、过滤不稳定问题，最终得到 2,723 个 cleaned problems。

其中 1,302 个问题拥有足够大的 duration spread，可以用 timing reward 区分不同实现。

数据被拆成两类 tests：

- correctness tests 用来判断程序语义是否正确
- optimization tests 使用更大的 workload，放大不同算法与实现之间的运行时间差异

这个拆分很重要。很多 coding benchmarks 的 tests 足以抓错，却短到无法稳定比较性能。一个程序快 10%，放到几十毫秒的执行里，差异很容易被系统噪声淹没。Input 规模扩大后，复杂度与实现质量才会形成可测量的间隔。

论文还比较了 local sandbox 与隔离的 remote execution service。训练 worker 上的本地执行会和 inference、rollout 争抢 CPU 与其他资源，使 timing 随训练负载漂移。Calibrated service 提供更稳定的参考环境，并允许把生成程序放入 human solution duration distribution 中排名。

Code optimization reward 的第一步，是让测量本身值得相信。

## Runtime 放在 reward 的哪个位置

即使 timing 已经稳定，直接把平均运行时间加到 correctness reward 里，效果仍然有限。论文报告，naive timing reward 对严格指标帮助很小，还会让 pure-correctness pass@1 下降 2 到 4 个百分点。

作者把 optimization constraint 分成三类介入位置。

### Pre-execution filtering

先筛选 optimization tests，再执行 candidate。筛选依据可以是 absolute duration、problem 内相对 duration，或者 input-output length。

这类方法能避开不稳定的 workload tails，对 human references 的依赖也较低。缺点是不同问题之间的性能压力较粗。

### Intra-execution limits

程序执行时直接施加 time limit。Limit 可以是全局阈值，也可以来自 strong human solutions。

这种约束非常直接，但阈值过严会让大部分 rollout 同时失败，reward 迅速变成全负值。GRPO group 内若所有 samples reward 相同，advantage 就失去训练信息。

### Post-execution ranking

先完整执行，再把 candidate 放入 calibrated human pool 中做 percentile ranking。

同一次 execution 可以应用不同 aggregation rules 与 percentile thresholds，实验灵活性更高。代价是需要可靠的 reference duration distribution。

所有环境最终向 reward 暴露三个量：

- correctness gate `c`
- optimization gate `g`
- optional graded quality `q`

比较稳健的基本原则是，incorrect solution 始终拿负 reward，只有通过 correctness gate 的程序才有资格获得 efficiency credit。这样可以减少 fast-but-wrong 的 reward path。

## 先用 offline simulator 筛 reward

论文探索的 environment 与 reward combinations 很多，而一次 online run 要使用 8 到 32 个 GPU nodes，持续数小时甚至数天。

作者先构建 offline simulator：

- 用 sampled human solutions 代替 model generations
- 用预先测量的 calibrated durations 代替 live sandbox calls
- 保留准备上线的 environment 与 reward computation
- 检查更好的 solutions 是否稳定得到更高 reward

Simulator 用 AUC 识别过于 sparse 或 saturated 的设置，用 curve steepness 与 monotonicity 判断 reward 是否真的区分 solution quality。

这个 simulator 不预测精确 learning curve。它承担的是 configuration triage，把明显 flat、sparse、saturated 或受噪声支配的 reward setup 提前排除。

这是一条很实用的 post-training 方法论。昂贵 RL run 之前，先问 reward 是否满足几个最低条件：

- 好样本的 reward 是否单调提高
- reward range 是否被大量全负或全正样本占满
- 测量噪声是否足以改变 solution ordering
- correctness 与 secondary objective 是否存在可利用的缝隙

## Sparse timing reward 怎样修改 GRPO

Timing-based optimization 比 correctness-only RLVR 产生更多 zero-advantage groups。论文在 Qwen 2.5 7B 的训练初期观察到，binary optimization reward 下这类 context 可以达到约 40% 到 50%。

Sandbox state 还会随 concurrent load 变化。两个不同时间采集的 reward，即使 policy 没变，也可能缺乏直接可比性。

作者保留 async-GRPO 主体，并做了几项稳定化修改。

### 增加 same-prompt rollouts

同一个 prompt 采更多 trajectories，可以提高 advantage 的 Monte Carlo estimate 质量，也能降低整组 samples 都拿相同 reward 的概率。

### 增大 trainer batch

更大的 batch 平滑 timing noise。论文报告，这项修改让 pure-correctness performance 提高 10% 到 25%，p50 optimization performance 最高提高 35%，p30 最高提高 60%。

### 只 center returns

训练对 group returns 做 centering，但不除以 group standard deviation。

标准差 normalization 可能放大只有微小 timing fluctuation 的 prompt，让噪声产生的 isolated reward difference 获得过高权重。真正发现多种 optimization tricks 的 group 反而可能被相对削弱。

### 固定 token horizon

作者使用 token-weighted prompt mean 作为 advantage baseline，并用固定的 32,768-token horizon normalize loss。

按每条 rollout 自己的长度 normalization，容易低估冗长错误轨迹的代价，也可能降低成功 long reasoning trajectory 的正向权重。Fixed horizon 让 token-level gradient 在不同 trajectory lengths 下保持更一致的尺度。

### 控制 reward staleness

训练会调节 worker/trainer ratio，让 generated batches 尽快被消费。Context 超过 30 optimizer steps 就丢弃，带来约 5% score gain。Replay buffer 也被移除，因为旧 reward 对应的 sandbox state 已经变化。

这里出现了一个很关键的系统结论：execution reward 带有采集时刻的环境状态。Queue、policy lag、sandbox load 与 replay policy 都会改变 reward semantics。

## 结果怎样看

论文使用 percentile-constrained pass@k。

`p100` 等价于 pure correctness。`p50` 要求程序正确，并进入 human reference leaderboard 的 top 50%。更小的 percentile 对速度要求更严格。

在 DMC-Optim 上，最强 optimization-aware configurations 将 strict p50 pass@1：

- Qwen 2.5 7B 从 18.0% 提高到 31.3%
- CWM 32B 从 30.7% 提高到 50.4%

在更严格的 p30 上，CWM 32B 的 relative improvement 达到 125%。这些收益没有牺牲 pure-correctness scores。

在 degraded timing sandbox 下，robust optimization RL 相对 standard RLVR 提高约 100% 到 200%，具体幅度取决于 evaluation criterion。LCB 上，CWM 32B 与 standard RLVR 的 median-sample speed comparison，最高赢得 83%。

和每题最快的 correct human submission 比，模型达到约一半的人类 complexity-class improvement rate，14% 对 28%。

## 为什么今天选它

这篇论文同时踩中 code capability、GRPO、reward design 与 distributed execution。

它给出的完整链路很少见：

- benchmark workload 决定速度差异能否被看见
- sandbox 决定测量是否稳定
- correctness gate 决定 reward 能否被投机
- simulator 决定哪些配置值得投入 online compute
- batch statistics 决定 noisy reward 怎样进入 gradient
- queue 与 staleness policy 决定不同 rollout 的 reward 是否可比

对于任何依赖真实系统指标的 RL，这套思路都值得复用。Latency、memory、energy、throughput 与 tool cost 都会遇到类似问题。Reward 进入训练之前，需要先建立稳定的 measurement protocol 与 temporal validity boundary。

## 另外两篇

第二篇是 **CodeNib: A Multi-View Data System for Serving Repository Context to Coding Agents**。

CodeNib 为每个 repository commit 建立 lexical、dense 与 structural views，统一返回 repository-relative source ranges，并在 edits 后维护 selected views。

在 100 个 snapshots 上，与完整 rebuild 结果一致时，graph update 与 vector update 的 median speedup 分别是 8.7 倍和 25.4 倍。Static navigation 在可对齐 live-server locations 的 subset 上快 4.7 倍。Across five models，context policies 在保持 localization 的同时，比 paired grep/read 少用 50% 到 87% trajectory tokens。

它把 repository context acquisition 做成一个拥有版本与 validity rules 的 data-system layer，能减少 coding agent 在长任务里的重复 discovery。

第三篇是 **Pass the Baton: Trajectory-Relayed On-Policy Distillation**。

这篇论文关注 on-policy distillation 的 prefix failure。Student 一旦早期走错 reasoning direction，后续 token 会沿错误路径继续生成，teacher 对这些 failed prefixes 的监督质量也会下降。

Relay-OPD 用 teacher-student continuation asymmetry 触发 label-free handoff。Teacher 在关键位置短暂接管，生成一小段 corrective leg，随后 student 恢复生成并接受训练。

用 Qwen3-4B-Instruct-2507 教 Qwen3-0.6B 与 1.7B Non-Thinking students 时，它在八个数学 reasoning benchmarks 上全部取得最好或第二好的结果。1.7B student 平均超过 standard OPD 5.73%，超过 FastOPD 1.49%，training trajectory length 减少超过 50%。

## 今日 3 篇精选

### 1) Reinforcement Learning for Code Optimization
- 链接: https://arxiv.org/abs/2607.25970
- 摘要速读: 用 DMC-Optim、calibrated sandbox、offline reward screening 与 stabilized async GRPO，让模型在保留 correctness 的同时学习生成更快代码。
- 为什么重要: Runtime reward 的质量由 measurement system、reward composition 与 rollout freshness 共同决定。

### 2) CodeNib: A Multi-View Data System for Serving Repository Context to Coding Agents
- 链接: https://arxiv.org/abs/2607.25431
- 摘要速读: 维护 repository 的 lexical、dense 与 structural views，统一服务 search、symbol navigation 与 bounded context。
- 为什么重要: Coding agent 的 context retrieval 可以成为独立、可版本化、可增量维护的数据系统。

### 3) Pass the Baton: Trajectory-Relayed On-Policy Distillation
- 链接: https://arxiv.org/abs/2607.26057
- 摘要速读: 在 student prefix failure 处让 teacher 短暂接管，再恢复 student rollout，用有限 teacher budget 修复高杠杆 reasoning branch。
- 为什么重要: 它提高 on-policy supervision 的有效密度，同时把 trajectory length 降低超过 50%。

## 一句话结论

今天最强的信号是：**当 RL reward 来自真实执行系统，measurement protocol、sandbox state、queue freshness 与 loss aggregation 都属于算法本身。**
