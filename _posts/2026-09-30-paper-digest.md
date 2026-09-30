---
title: "Paper Digest: 2026-09-30"
categories: [Paper Digest]
tags: [AI, LLM, Post-training, Reinforcement Learning, SFT, Code Agents]
---

今天最值得看的 paper 是 **ROSS: Relearning from Self-Generated Rollouts through Selective Supervision**。

LLM post-training 会消耗大量算力生成 rollouts。Policy 更新以后，这些历史轨迹通常被视为 stale data：它们来自旧模型，分布已经落后，整条模仿还可能把错误、无效尝试和冗余动作重新教给新 checkpoint。

ROSS 提出一个很实用的问题：**已经付过采样成本的 self-generated experience，能否继续训练后续 policy？**

答案取决于怎样使用轨迹。历史 rollout 里往往同时存在三类内容：

- 后续 checkpoint 已经遗忘或无法稳定复现的有效行为
- 探索过程中的错误和 abandoned attempts
- 对结果没有帮助的重复动作

把整条轨迹当成普通 SFT target，会无差别地学习这些内容。ROSS 保留完整历史轨迹作为 context，只在经过筛选的 model-generated continuations 上计算 loss。模型能看到一个决定发生前的全部过程，同时只对值得保留的行为接受监督。

## Full context，selective loss

ROSS 的关键设计可以压缩成一句话：**context 不裁剪，supervision 要选择。**

保留 full trajectory 很重要，因为一个 continuation 是否合理，经常取决于此前已经观察到的证据、调用过的工具、失败过的尝试和当前任务状态。如果只截取最终答案，训练数据会丢失行为发生的条件。

Selective supervision 则避免把旧 rollout 当作完美 demonstration。Loss mask 只覆盖选中的 continuation spans，其他 tokens 继续提供上下文，却不会成为需要复制的 label。

这让 historical experience 的角色更接近 replay buffer：它保留曾经出现过的有价值行为，再通过选择机制控制哪些部分能够影响新 policy。

## 三种 post-training 场景

论文在三个设置中验证 ROSS：

1. domain-specific reinforcement learning
2. multi-teacher on-policy distillation
3. agentic reinforcement learning

覆盖的能力包括 mathematics、code generation、instruction following 和 software engineering。结果显示，ROSS 能持续改善上游 checkpoints，并超过直接使用历史数据的基线。

在 Qwen3.6-35B-A3B 上，六个 benchmark 的 multi-teacher OPD 平均成绩从 **58.40% 提升到 62.20%**。在更接近真实代码智能体的 SWE-bench Verified 上，成绩从 **64.20% 提升到 68.40%**。

这些提升来自 offline SFT，不需要再让当前 policy 生成新 rollouts。对于 rollout 成本很高的 agentic tasks，这一点尤其有价值。

## Rollout 是可以折旧的训练资产

很多 post-training pipeline 把 rollout 看成当前 update 的一次性输入。ROSS 提醒我们，轨迹库本身可能是一项可复用资产。

一条成功轨迹的价值不限于最终 reward。它还包含：

- 哪些状态下出现了关键决策
- 哪些工具调用真正推动了任务
- 哪些恢复动作让 agent 从失败中返回
- 哪些行为后来被 policy 遗忘

只要保存完整 trajectory、checkpoint identity、reward、verifier evidence 和 span-level metadata，后续训练就能重新判断哪些经验仍然兼容、哪些行为值得恢复。

这也改变了 rollout infrastructure 的衡量方式。系统除了追求 samples per second，还应考虑每个 sample 在未来能被复用多少次，以及选择机制能否把旧数据中的有效行为与噪声分离。

## 对 code-agent post-training 的启发

Code agents 很适合测试 ROSS，因为行为选择可以获得丰富证据：unit tests、integration tests、patch diff、tool outcomes、review signals 和 task completion state。

一个直接的实验可以复用历史 code-agent trajectories，比较四种训练方法：

- full-trajectory SFT
- final-answer 或 final-patch SFT
- ROSS-style selected-span SFT
- 重新进行 fresh on-policy rollouts

Span selector 可以从确定性规则开始，例如只监督通过 targeted tests 后的修复动作，或者保留从失败状态走向有效 patch 的 continuation。更进一步，可以让 verifier 或 grader 评估某个 action 是否必要、是否仍与新 checkpoint 兼容。

除了 SWE-bench solve rate，还应追踪 retained capability、trajectory length、工具调用次数、错误行为回流，以及每一点提升对应的新 rollout tokens。真正关键的问题是：**过去花在探索上的算力，有多少能够沉淀为后续模型继续使用的 behavioral memory？**

## 今天另外两篇相关论文

**Learning Beyond What You Sample** 提出 GRAFT，在 GRPO 出现 all-fail groups 时，用异构 peer model 的轨迹补充训练信号。它通过 sequence-level compatibility weighting 和 token-level importance-ratio clipping 控制 off-policy mismatch，在相同 per-model rollout budget 下平均提升 2.1 points。

**EasyPPO** 把 LLM PPO 的不稳定定位到 critic。它让 critic 保留 truncated rollouts 的 returns，只在 actor 侧过滤 overlong samples，同时按 prompt return variance 归一化 critic regression，并用较小 critic mini-batches 限制 outlier 影响。在 coding、math 和 multi-turn search 上都优于 vanilla PPO。

论文：<https://arxiv.org/abs/2609.35954>

另外两篇：<https://arxiv.org/abs/2609.37868>、<https://arxiv.org/abs/2609.36802>
