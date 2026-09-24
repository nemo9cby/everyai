---
title: "Paper Digest: 2026-09-24"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Code Agents, Reinforcement Learning, Post-training]
---

今天最值得看的 paper 是 **PACT: From Credit Assignment to Critic Alignment**。

LLM agent 的一条 trajectory 可能有几万 tokens、几十次 tool calls，训练时却常常只在最后拿到一个 verifier reward。这个结果该怎样分给前面每个 token，一直是 agent RL 里最难处理的问题之一。

PACT 先给 token-level credit 一个严格定义，再根据这个定义检查现有算法，最后落到一个很小但有效的训练流程改动。它在四个数学推理 benchmark 上达到 72.87% Avg@16，在 SWE-bench Verified 上达到 67.4% pass@1。

## Token credit 的唯一形式

设 `F_i` 表示模型生成第 i 个 token，并看到对应环境反馈后的全部信息。论文定义此时对最终 reward 的预测为：

`V_i = E[R | F_i]`

作者提出三个条件：

- Completeness：所有 token 的 credit 之和，要解释最终 reward 相对初始预测的变化
- Prefix Consistency：已经生成的 prefix，其累计 credit 不能被未来不同的分支改写
- Neutrality：在生成下一个 token 之前，它的期望 credit 应为零

满足这三个条件时，token-level credit 存在唯一解：

`C_i = V_i - V_{i-1}`

直觉很简单。一个 token 的 credit，就是它和随后的环境反馈出现以后，我们对最终成功概率的判断改变了多少。Turn-level credit 也可以通过累加一段连续 tokens 得到。

这个结论还统一解释了几个常见现象。理想的 on-policy distillation teacher 会产生与真实 token credit 成比例的 policy-gradient update，相当于一个隐式 critic。RLOO 虽然把 response-level signal 广播给所有 tokens，它在期望上仍能得到与唯一 credit 相同的梯度贡献，只是方差可能更大。

## 长 trajectory 为什么让 critic 更脆弱

论文证明，在 reward 被归一化到 `[0, 1]` 后，所有 token credit 的平方和期望最多为 `1/4`。Trajectory 变长时，真正显著的局部 credit 依然很少。

这会放大 critic error。GAE 使用 `lambda < 1` 时，中间每一步的 value estimation error 都会进入 advantage。大量很小的真实 credit 容易被这些误差覆盖。`lambda = 1` 会消掉中间 critic error，只留下当前 prefix 的 value error。实验中，PPO 的 `lambda = 0.95` 出现 policy collapse，`lambda = 1` 明显更强。

还有一个更直接的系统问题：policy-critic lag。

传统 PPO-style 流程用 policy `π_k` 收集 rollout，critic 往往仍主要拟合上一轮 policy。Actor 更新成 `π_{k+1}` 后，critic 又落后了一次更新。对于依赖相邻 value 差值的 token credit，这个版本错位会成为系统性噪声。

## PACT 怎样对齐 critic

PACT 采用 Actor-then-Critic 顺序：

1. 用当前 critic 为 rollout 计算 actor update 所需的 values
2. 完成 actor update
3. 用更新后的 actor 对同一批 rollout 再做一次 forward pass
4. 根据更新前后的 token probabilities 计算 importance ratio
5. 用 importance-corrected targets 训练 critic，使它对齐更新后的 policy

Critic 使用 sigmoid 输出与 binary cross-entropy。对于 `[0, 1]` reward，BCE 和 MSE 的最优预测都是 conditional mean，但论文的受控实验中 BCE 收敛更快，成功与失败 trajectory 的 value separation 也更大。

精确 continuation importance ratio 在长序列上方差太高，实际实现使用 detached current-token ratio，并屏蔽 ratio 超出范围的 critic loss。额外成本是一遍 actor forward，不需要新增 rollout。

## 实验结果

数学推理使用 Qwen3.5-4B，在 AIME 2025、AIME 2026、BeyondAIME 和 HMMT Nov. 2025 上评测：

- Base model：41.01%
- GRPO：64.07%
- PPO，`lambda = 1`：59.71%
- PACT without importance sampling：67.74%
- PACT：72.87%

Importance correction 单独贡献了 5.13 个百分点。

Coding agent 实验使用 Qwen3.6-35B-A3B，在 OpenSWE 上训练，通过 Codex agent 与环境交互，SWE-bench Verified pass@1 为：

- Base model：60.8%
- GRPO：65.4%
- PPO，`lambda = 1`：65.0%
- SAO：63.6%
- PACT：67.4%

Coding 上的提升比 math 小，但它来自很具体的 critic synchronization 改动，并且保持相同的 terminal verifier reward。

## 对 agent post-training 系统的启发

Policy 与 critic 的版本关系应该成为一等监控指标。每条 trajectory 至少记录：生成它的 actor checkpoint、计算 advantage 的 critic checkpoint、actor update 后的 policy version，以及 critic target 实际对齐的版本。

在异步和异构训练里，这个问题会更尖锐。Rollout workers、training workers 和 critic service 有各自的更新节奏，表面上的 on-policy batch 也可能携带多层 version skew。PACT 给出了一个适合做 controlled experiment 的起点：固定 rollout、batch、reward 和 actor-side clipping，只改变 Actor/Critic 更新顺序、BCE objective 与 importance correction，观察收益来自哪里。

论文当前的主要限制是实验规模。Coding 只展示一个模型与 SWE-bench Verified，数学实验也集中在 outcome-verifiable tasks。它是否适用于更噪声的 reward model、多轮开放式任务和高度异步的 agent RL pipeline，还需要复现。

## 今天另外两篇相关论文

**Schrödinger's Code Repository** 对 SWE-bench repository 做行为保持的动态变换，包括 namespace remapping、文件内布局重排和 code rewriting。模型性能下降、交互成本上升，额外消耗主要出现在 repository exploration 与 localization。这给 code-agent evaluation 提供了一种 metamorphic robustness test。

**Just-in-Time Memory** 保留 raw trajectories，把 memory compression 推迟到新 query 到来之后。Query-conditioned curator 的输出会立刻被 executor 使用，因此可以直接用当前任务 reward 做 GRPO。它比最强 write-time memory baseline 在 ALFWorld、WebShop 和 tau2-bench 上分别提高 16.2、16.3 和 3.9 个百分点。

论文：<https://arxiv.org/abs/2609.26355>

另外两篇：<https://arxiv.org/abs/2609.27891>、<https://arxiv.org/abs/2609.27334>
