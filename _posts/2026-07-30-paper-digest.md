---
title: "Paper Digest: 2026-07-30"
categories: [Paper Digest]
tags: [AI, GRPO, Reinforcement Learning, Credit Assignment, Post-Training, Code Agents]
---

今天最值得看的 paper，我会选 **CoRT: Counterfactual Replay for Token-Level Rubric-Guided Policy Optimization**。

Rubric-based RL 有一个容易被忽略的信息损失。

Verifier 原本可以逐条判断 response 是否满足 correctness、formatting、safety、style 等 criteria。进入 GRPO 后，这些结构化判断通常会聚合成一个 scalar reward，再变成 response-level advantage。计算 policy loss 时，同一个 advantage 被广播给整条 response 的所有 token。

一条回答拿到正 advantage，开场铺垫、关键结论、格式标记和无关 filler 都得到同方向、同强度的更新。Rubric 知道模型满足了哪些要求，optimizer 却不知道这些要求具体落在哪些 token 上。

CoRT 提出一个很轻的解决方案：保持 rollout、verifier 与 response-level reward 原样，只增加一次 fixed-response counterfactual scoring。

同一条已经生成的 response，分别放在两个 context 下重新计算 token log probabilities：

- `x+` 包含原始 instruction 与 rubric criteria
- `x-` 只保留 instruction，移除 criteria

如果某个 token 在 `x+` 下明显更可能出现，说明它的生成更依赖 rubric context。两次 log probability 的差值可以作为 token-level rubric dependence 的 proxy。

这条 signal 不需要额外 generation，不需要再调用 verifier，也不需要训练一个 token relevance model。

## GRPO 丢掉了哪部分信息

对同一个 prompt，GRPO 采样一组 responses，并根据 reward group 计算：

`A_i = (r_i - mean(r)) / (std(r) + epsilon)`

之后，response `i` 的每个 token 都使用同一个 `A_i`。

这种设计简单、稳定，也适合 outcome-verifiable tasks。当 reward 只表示最终答案对错时，模型很难得到更细的 attribution，uniform broadcast 至少保持了清楚的优化目标。

Rubric reward 的结构更丰富。假设要求是：

1. 提到 reinforcement learning
2. 解释 group-relative advantage
3. 控制在 80 words 内

Verifier 可以逐项检查。聚合后的 reward 只能告诉 optimizer 整体表现，无法指出哪些 spans 对应概念解释，哪些 token 负责长度控制，哪些内容与 criteria 关系很弱。

CoRT 关注的就是这段从 criterion-level judgment 到 token-level update 的断层。

## 用 criteria-free prompt 做 counterfactual replay

CoRT 先按普通 GRPO 生成 response：

`y_i ~ pi_old(. | x+)`

正常 rollout 已经给出每个生成 token 在 full prompt 下的 log probability：

`l+_i,t = log pi(y_i,t | x+, y_i,<t)`

然后固定 response token sequence，不做新的 sampling，只把 prompt 换成 criteria-free 版本，再跑一次 scoring：

`l-_i,t = log pi(y_i,t | x-, y_i,<t)`

两者相减：

`delta_i,t = l+_i,t - l-_i,t`

这里 prefix 与 target token 都完全相同，唯一变化是 rubric criteria 是否存在。

正 contrast 表示这个 token 在 rubric context 下获得了更高 likelihood。接近零表示 criteria 对它影响较小。负 contrast 表示移除 criteria 后，这个 token 反而更容易出现。

论文把 `delta` 称为 rubric dependence 的 proxy。这个措辞很克制。Log-probability contrast 没有直接证明某个 token 对最终 reward 的因果贡献，它衡量的是 policy 对 criteria context 的局部敏感度。

这个区别很重要，也决定了方法的适用边界。

## 怎样把 contrast 变成稳定的 token credit

Raw log-probability contrast 可能 heavy-tailed，也会随 model scale 和 token distribution 变化。CoRT 用 sigmoid 把它映射到 bounded score：

`s_i,t = sigmoid(tau * (delta_i,t - b)) - 1/2`

然后构造 provisional weight：

`w~_i,t = 1 + eta * lambda_k * s_i,t`

其中 `eta` 控制 weighting strength，`lambda_k` 控制训练进行到第 `k` step 时的启用程度。

作者没有从第一个 update 就把 token weighting 全量打开，而是使用 SmoothStep schedule。Warmup 之后，`lambda_k` 从 0 平滑增长到 1，并在两个端点保持零斜率。

最后，每条 response 内部再做一次 normalization：

`w_i,t = w~_i,t / mean_t(w~_i,t)`

因此每条 response 的 token weights 平均值始终为 1：

`mean_t(w_i,t) = 1`

Token-shaped advantage 写成：

`A_hat_i,t = stop_gradient(w_i,t) * A_i`

这个约束保留了一个关键性质：整条 response 的平均 advantage 仍然等于原来的 `A_i`。CoRT 改变 credit 在 token 之间的分布，不改变 verifier reward，不改变 group-relative normalization，也不改变这条 response 应该被 reinforce 还是 suppress。

正 advantage 下，rubric-dependent tokens 得到更强强化。负 advantage 下，它们也会得到更强抑制。

## 为什么 normalization 与 schedule 都需要

论文的 ablation 很实用。

去掉 response normalization 后，mean token weight 会逐渐漂到 1 以上。看起来只有几个百分点，实际效果相当于给 response-level advantage 加了一个随训练变化的 multiplier。训练后期出现更高的 length clipping、gradient norm 与 entropy。

保留 normalization、去掉 SmoothStep ramp 后，平均权重尺度仍然受控，但 token weighting 在早期突然介入，后期同样会出现 gradient 与 entropy spike。

两个组件解决不同问题：

- response normalization 控制 update scale
- scheduled activation 控制 token redistribution 介入的速度

完整配置在 500 steps 附近保持较低 length clipping、稳定 actor gradient norm 和更好的平均 validation。

这个结果提醒我们，token-level credit shaping 不能只看 attribution signal 是否合理。它还会改变 optimizer 看到的 coefficient distribution。Scale、activation timing 与 clipping behavior 都属于算法的一部分。

## 只增加一次 scoring pass

CoRT 的工程成本相对清楚。

Full-prompt log probabilities 已经来自 rollout。新增部分是把同一 response 放到 criteria-free prompt 下，做一次 teacher-forced forward scoring，得到逐 token log probabilities。

它不需要：

- 额外采样 response
- 额外 verifier 或 LLM judge call
- rejection sampling
- token-level relevance labels
- 单独训练和部署 relevance discriminator

相比 learned token relevance 方法 RTT，CoRT 把额外复杂度放在 policy 自己的一次 replay scoring 上。

长 response 下，这次 forward pass 依然有明显成本。它省掉的是 generation 与外部 supervision pipeline，并没有让 token-level attribution 变成免费信号。实际系统需要测量 scoring throughput、KV reuse 空间，以及 rollout 与 learner 之间怎样传递 full-prompt log probabilities。

## 实验覆盖了什么

训练使用 HIR-16k。每个样本包含原始 prompt 和 instruction list，因此可以自然构造 full-rubric 与 criteria-free 两个 context。

Reward 使用两种 granularity：

- CSR，按满足 criteria 的比例给分
- AON，只有全部 criteria 都满足才给 1

主实验覆盖 Qwen3-4B-Instruct 与 Qwen2.5-7B-Instruct，并在 Qwen3-14B 上验证 scale。评测包括 IFEval、IFBench、MultiDimIF 与 AdvancedIF。

论文报告，CoRT 相对 matched response-level GRPO 的平均提升为 4.4 percentage points。

几个具体结果：

- Qwen3-4B + CSR 在 MultiDimIF 上从 74.38 提升到 80.48
- Qwen3-4B + AON 在 MultiDimIF 上从 74.25 提升到 81.60
- Qwen2.5-7B + CSR 在 MultiDimIF 上从 66.67 提升到 78.52
- Qwen2.5-7B + AON 在 MultiDimIF 上从 69.41 提升到 79.63

提升并非每个 metric 都一致。Qwen2.5-7B + AON 在 IFBench 的 prompt 和 instruction metrics 上分别下降 2.31 与 1.91 points。Qwen3-14B + AON 在 IFEval 上也略低于 GRPO。

这些 exceptions 值得保留。Counterfactual weighting 对 criteria-rich、multi-dimensional evaluation 看起来特别有效，在 sparse AON reward 与不同 benchmark 上仍会出现 metric-specific tradeoff。

论文还把 CoRT 接到 DAPO 与 GSPO 上。加入 CoRT 后，DAPO 的五项指标全部提升，GSPO 五项中四项提升。因为 credit redistribution 发生在 response-level advantage 计算之后，它可以和不同 policy objectives 组合。

## 和 learned token relevance 的差别

RTT 训练一个 token-level relevance discriminator，把 rubric feedback 转成 token credit。

CoRT 使用 policy-internal signal：同一 token 在有无 criteria 时的 likelihood contrast。

两条路线的 tradeoff 很直接。

Learned relevance model 可以吸收额外 supervision，理论上也能学习更接近 task outcome 的 attribution。代价是独立训练阶段、额外模型组件、label construction 与 serving integration。

Counterfactual replay 复用现有 policy，部署更紧凑。它依赖一个假设：policy 对 rubric prompt 的 likelihood sensitivity 与值得分配 credit 的 token 有足够强的相关性。

主实验里 CoRT 和 RTT 竞争力接近，很多设置下更强。这个结果说明，在 rubric-conditioned instruction following 里，额外训练 relevance model 并非获得 token credit 的唯一途径。

## 我最关心的三个问题

第一，criteria-free prompt 的构造是否稳定。

HIR-16k 天然提供 prompt 与 instruction list 的分离。在真实 post-training data 中，rubric 可能混在 system prompt、user request、policy constraints 和 hidden grader specification 里。移除哪些字段、保留哪些 context，会直接影响 `delta` 的含义。

第二，contrast 衡量的是 rubric dependence，reward contribution 还需要验证。

一个 token 可能高度依赖 criteria，却并未真正提高 task reward。模型也可能用很分散的 representation 满足某条 requirement，使关键 span 没有明显 contrast。Matched controls、criterion-wise removal 和 span analysis 会决定这条 signal 能否迁移到更复杂任务。

第三，它能否进入 code RL。

Coding tasks 经常同时包含 correctness、API usage、complexity、format、security 与 style criteria。若把 rubric 拆成显式 constraints，CoRT 可能帮助模型把 credit 集中到 import choice、boundary checks、algorithm selection 或 output format 等局部结构。

代码 token 之间有更强的长程依赖。一个早期 function signature 会影响后续整段实现，单 token likelihood contrast 能否对应真实 execution outcome，需要专门实验。

## 为什么今天选它

CoRT 的价值在于边界很干净。

它没有重写 GRPO，也没有要求新的 reward source。它保留现有 rollout、rubric verifier、group-relative advantage 与 clipped policy objective，只在 advantage broadcast 之前增加一次 policy-internal counterfactual replay。

方法本身可以用几行公式讲清楚，工程实现也容易定位：

1. 保存 full-rubric rollout log probabilities
2. 对 fixed responses 做 criteria-free scoring
3. 计算 bounded token weights
4. 在 response 内 normalize
5. 按 schedule 逐步启用
6. 将 weights 乘到 signed GRPO advantage

对于正在搭建 post-training system 的团队，这类 work 很有吸引力。它提供一个可以独立 ablate、独立 profile、独立回滚的 credit-assignment module。

## 另外两篇

第二篇是 **MindForge: Teaching Small Language Models Whole-Life-Cycle Software Engineering via Source-Free Program Synthesis**。

MindForge 把开源 command-line programs 转成 source-free environments。Agent 只能看到 documentation 与 compiled reference executable，需要通过行为探索重建完整程序。

作者用与 ProgramBench repositories 不重合的环境生成 GLM-5.2 teacher trajectories，fine-tune Qwen3.6-27B 后，ProgramBench average test pass rate 从 37.98% 提高到 49.51%。

更有意思的是 cross-benchmark transfer。模型在 repository generation、translation、bug fixing、feature implementation 与 multilingual issue resolution 等七个 unseen benchmarks 上全部提升，最高增幅达到 31 points。

它给 code SFT 一个很实用的数据方向：训练完整 software lifecycle，包括 requirements discovery、oracle probing、implementation 与 validation，而不局限于现有 repository 上的 patch task。

第三篇是 **SkillRise: Agentic Reinforcement Learning for Cross-Task Skill Evolution**。

SkillRise 让同一个 policy 在 solving tasks 与维护 skill document 之间交替。Related tasks 被排列成逐步变难的 sequence，当前 task 生成的 skill document 会直接传给下一项任务。

Solving action 使用当前 task outcome，skill curation 使用 discounted downstream outcomes。一个 skill 是否有价值，由它对后续 tasks 的影响决定。

在 ALFWorld、WebShop 与 ScienceWorld 上，SkillRise 相对最强 baseline 提高 2.3 到 8.5 percentage points。即使每个 task 只尝试一次，sequence 越长，performance 仍持续提高。

这篇对 agent memory 与 skill systems 很有启发：persistent skill artifact 可以成为 policy action，未来 task outcome 可以成为训练它的 reward。

## 今日 3 篇精选

### 1) CoRT: Counterfactual Replay for Token-Level Rubric-Guided Policy Optimization
- 链接: https://arxiv.org/abs/2607.25659
- 摘要速读: 对同一条 response 做 full-rubric 与 criteria-free scoring，用 token log-probability contrast 重分配 GRPO advantage。
- 为什么重要: Rubric 的结构化信息可以进入 token-level credit，同时保留原有 reward 与 group-relative update scale。

### 2) MindForge: Teaching Small Language Models Whole-Life-Cycle Software Engineering via Source-Free Program Synthesis
- 链接: https://arxiv.org/abs/2607.27146
- 摘要速读: 把开源 CLI programs 变成只暴露 documentation 与 executable oracle 的 source-free synthesis environments。
- 为什么重要: 完整程序重建 trajectories 能同时训练 requirements、exploration、implementation 与 testing，并迁移到多类 SWE tasks。

### 3) SkillRise: Agentic Reinforcement Learning for Cross-Task Skill Evolution
- 链接: https://arxiv.org/abs/2607.26784
- 摘要速读: 一个 policy 同时解决任务和维护 persistent skill document，用未来任务结果训练 skill curation。
- 为什么重要: Cross-task skill reuse 获得了显式 credit-assignment path。

## 一句话结论

今天最强的信号是：**Rubric 已经包含结构化 supervision，训练系统需要保留它在 response 内部的作用位置。**
