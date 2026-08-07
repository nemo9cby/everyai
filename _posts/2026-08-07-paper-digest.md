---
title: "Paper Digest: 2026-08-07"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, Post-Training, Credit Assignment, Code Agents]
---

今天最值得看的 paper，我会选 **AgentOPSD: Recursive Self-Distillation for Agentic Reinforcement Learning**。

它处理 long-horizon agentic RL 里一个很具体的问题：trajectory 结束后只有一个 success reward，训练系统怎样知道中间哪些 turn 真正改变了结果？

GRPO 会先根据同一问题的一组 rollouts 计算 trajectory-level advantage，再把它广播给轨迹里的所有 tokens。几十轮交互中，关键决策、常规操作、无效绕路和偶然错误因此共享同一个方向与强度。轨迹越长，这种 credit ambiguity 越严重。

AgentOPSD 的思路是维护一个关于“这条轨迹最终会成功”的 belief state。每个 action turn 都提供一份新 evidence，系统测量它让 success belief 改变了多少，再用这个变化重分配 GRPO advantage。

## Self-teacher 提供 turn-level evidence

论文使用同一个 policy 构造 student 与 privileged self-teacher。

Student 看到正常的 interaction history。Self-teacher 额外看到一条 training-only skill，其中描述了可能有用的 subgoals 和 action patterns。这些 skills 来自 SkillRL 的 SkillBank，通过 keyword matching 检索，推理阶段不会使用。

对 student 已经生成的每个 token，系统分别计算 teacher 与 student 的 log probability，并取两者之差：

`δ(k,t) = log π(y(k,t) | history, skill) - log π(y(k,t) | history)`

一个 action 通常由多个 tokens 组成。AgentOPSD 在 turn boundary 将这些 token gaps 求和：

`e(k) = Σ_t δ(k,t)`

正值表示这个 action 在 privileged skill 条件下更容易出现，可以视为支持成功的 evidence；负值提供相反信号。这里的聚合粒度很重要，environment response 对应完整 tool call 或 action，而非单个 token。

## 递归更新 success belief

局部 gap 还不足以表达 sequential credit。同一个 action 在早期可能是关键突破，到了证据已经充分的后期则可能只是重复确认。

AgentOPSD 用 GRPO group 的平均成功率初始化 belief prior：

`B(0) = clip(group success rate)`

随后在 log-odds space 递归累计 turn evidence：

`c(k) = γ c(k-1) + e(k)`

`B(k) = sigmoid(logit(B(0)) + c(k))`

论文使用 `γ = 0.95`，让很早以前的 evidence 逐渐衰减。每一轮的 credit 来自 belief revision：

`ΔB(k) = B(k) - B(k-1)`

这个设计会自动压低冗余 evidence。当 belief 接近 0 或 1 时，sigmoid 进入饱和区，相同大小的局部 gap 对 belief 的影响更小；当结果仍不确定时，同样的 evidence 会造成更明显的 revision。

## 用 verifier 决定方向，用 belief revision 决定强度

论文没有让 self-teacher 取代 outcome verifier。Terminal reward 仍然决定整条 trajectory 的优化方向。

AgentOPSD 将 `ΔB(k)` 与 trajectory advantage 的符号对齐，再在每条轨迹内部做标准化。得到的 turn credit 被映射为一个有界且始终为正的 multiplier，用于缩放原始 GRPO advantage：

`A_turn(k) = A_sequence × [(1 - λ) + λ w(k)]`

主实验使用 `λ = 0.5`。由于 multiplier 保持正数，一个 turn 只会被放大或减弱，不会把 verifier 给出的更新方向翻转。每个 turn 内的 policy tokens 继承同一个 reshaped advantage，随后进入标准 clipped policy objective。

整个方法没有额外 distillation loss，也不训练 critic。额外开销来自 skill retrieval，以及同一批 student actions 在 privileged branch 上的概率计算。Rollout 数量保持不变，推理阶段不需要 skills 或 teacher branch。

## 实验结果

作者在 ALFWorld、WebShop 和 Search-QA 上训练 Qwen2.5-3B/7B-Instruct，使用 8 张 H800。所有 self-distillation baselines 共享同一组 privileged skills、数据和 training budget，主要差别落在 teacher-student gap 如何进入优化。

Qwen2.5-7B 在 ALFWorld 上达到 89.1% success rate。机制消融更能说明方法贡献：

- 完整 AgentOPSD：89.1%
- 改成 per-token accumulation：85.9%
- 用 raw local gap 代替 recursive belief revision：82.8%
- 只保留 revision magnitude，去掉 outcome-aligned sign：80.5%
- 去掉 group success rate 提供的 belief prior：78.9%

论文还按任务平均交互轮数回归 performance degradation。每增加一个 turn，GRPO 平均损失 2.91 个 success points，RLSD 损失 3.59 个点，AgentOPSD 只损失 0.54 个点。这个结果正好对应方法的目标区间：当 interaction horizon 变长，history-dependent credit 的价值明显增加。

Hyperparameter sweep 显示 `λ` 最重要。`λ = 0.5` 的结果最好，继续减小会削弱 turn-level reshaping。Evidence decay `γ` 在 0.8 到 1.0 之间没有单调趋势，说明收益主要来自递归状态与 marginal revision 本身，对具体衰减速度相对稳定。

## 对 agentic post-training 的启发

这篇 paper 最值得保留的抽象是：terminal verifier 提供 global direction，privileged evidence 负责在 trajectory 内部分配 gradient strength。

这个结构很适合 code agents。Passing tests 可以继续作为 outcome verifier。Training-only branch 可以额外读取 issue decomposition、reference invariants、execution trace summaries 或检索到的 repair skills。Teacher-student likelihood gap 先按 patch、tool call 或 action turn 聚合，再通过递归 belief revision 找出真正改变成功概率的步骤。

它也避免了 counterfactual rollout 的成本。要直接估计某一步的 causal contribution，通常需要从同一中间状态采样多条 continuations；AgentOPSD 用 self-distillation evidence 构造近似信号，把额外成本压到 scoring forward passes。

几个限制也很清楚。Bayesian belief 是一种解释框架，self-teacher gap 只是 ideal Bayes factor 的 proxy，`B(k)` 不能当作经过校准的真实成功概率。方法依赖 privileged skills 的覆盖率和质量，keyword retrieval 错误会直接污染 credit。实验 backbone 仍是 Qwen2.5，环境以 text agent tasks 为主，迁移到代码仓库和 GUI agent 时需要重新验证 turn boundary、skill representation 与 noisy verifier 的相互作用。

## 为什么今天选它

AgentOPSD 给出的改动很小，却抓住了 agentic RL 的核心结构：tokens 组成 action，actions 改变 environment state，trajectory 最后才得到 reward。它把 self-distillation signal 对齐到 action boundary，再用递归状态把局部 evidence 变成 sequential credit。

对于已经有 GRPO rollout、verifier 和 training-only knowledge 的系统，这是一条可以直接试验的路线。实现上只需增加 privileged scoring branch、turn aggregation 和 advantage reshaping，无需新增 critic 或扩大 rollout budget。

## 另外两篇

第二篇是 **EnvACE: Internalizing Environment Dynamics via World Rehearsal for Agentic Reinforcement Learning**。

EnvACE 让 policy 在生成 tool call 后继续扮演 environment，预测这次 action 会得到什么 response，再基于自己 rehearsed 的 observation 规划下一步。Actor 与 environment-modeling role 一起用 task success reward 优化。论文在 BFCL-v4、tau^2-Bench、VitaBench 和 FinMCP-Bench 上报告了可迁移收益，test-time 还能在真正执行前做 private rehearsal。

它对 distributed rollout system 的价值很直接：外部 environment 往往是 throughput bottleneck，world rehearsal 可以把一部分交互成本改写成模型 inference。随之而来的风险是 simulated response error 会在长链里累积，grounding 与 calibration 会决定这条路线能扩多远。

第三篇是 **CalibForge: Adversarial Solver Calibration for Scaling Learnable Terminal Tasks**。

CalibForge 用 solver behavior 反向修改生成的 terminal tasks。Multi-solver calibration 寻找不同 solvers 产生分歧的任务，contrastive calibration 寻找 strong solver 能通过、weak solver 会失败的任务。目标是构造 solver-relative learnable zone，让训练数据保持可解，同时对当前能力仍有挑战。

作者生成 5,431 道 calibrated tasks。用完整数据训练后，相对对应 base model，Terminal-Bench 2.0 最大提高 24.71 points，SWE-bench Pro 提高 27.68 points，Doc2Repo 提高 30.04 points。对 code-agent data pipeline 来说，这是一种很实用的 curriculum signal：task validity 由执行验证保证，task utility 由 solver discrimination 保证。

论文：<https://arxiv.org/abs/2608.05987>

另外两篇：<https://arxiv.org/abs/2608.06197>、<https://arxiv.org/abs/2608.06352>
