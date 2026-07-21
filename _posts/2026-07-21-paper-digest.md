---
title: "Paper Digest: 2026-07-21"
categories: [Paper Digest]
tags: [AI, Reinforcement Learning, Post-Training, Distillation, Code Agents]
---

今天最值得看的 paper，我会选 **Distilled Reinforcement Learning for LLM Post-training**。

它处理的是 reasoning model post-training 里一个很实际的冲突。

RL 依赖 outcome reward，优化目标清楚，但 long response 上的 token-level credit assignment 很稀疏。On-policy distillation 可以让 teacher 给每个 token 提供 dense signal，但 conventional OPD 会用 KL divergence 持续拉近 student 与 teacher 的整套分布。当两者属于不同 model family、reasoning style 差异较大时，这种 unconditional imitation 很容易提前收敛，也可能干扰 reward optimization。

Distilled RL 把 teacher supervision 直接放进 RL objective，用 teacher 对 student-generated tokens 的偏好重新加权 advantage。方法包含三个组件。

第一，reverse importance sampling。Teacher 与 student 对每个 sampled token 的概率比会成为 learning signal 的权重，并通过 clipping 控制方差。

第二，negative sample reset。只要一条 trajectory 的 advantage 为负，teacher weight 就重置为 1。这个细节很关键。负样本已经会被 policy gradient 压低概率，如果继续按 teacher preference 放大其中某些 token 的惩罚，student 可能被推离 teacher distribution，dense supervision 反而带来负迁移。

第三，sequence-level geometric normalization。它校准一条 response 内大量 token ratio 的整体尺度，避免 teacher 普遍给 student tokens 较低概率时，所有更新都被系统性缩小。

作者在 EasyR1 和 VeRL 中实现方法，用 Qwen3-8B-GRPO 作为 teacher，在 DAPO-17K 上训练 Qwen3-4B、Qwen3-1.7B 和 DeepSeek-R1-Distill-Qwen-1.5B。

结果相当整齐。

在 cross-family 的 DSQW-1.5B 上，十个 math benchmarks 的平均 pass@1 由 31.70 提升到 40.00。Distilled RL 分别超过 OPD、GRPO 和 OPD+RL 4.73、3.14 和 3.46 points。

Qwen3-4B 的平均分由 46.33 提升到 58.96，比 OPD 高 2.99 points，比 GRPO 高 1.56 points。Qwen3-1.7B 上的领先幅度较小，但仍然保持第一。MMLU-Pro、SuperGPQA 和 pass@16 结果也显示提升没有局限在训练时的数学 reward。

最有信息量的是 ablation。去掉 negative sample reset 后，Qwen3-4B 和 DSQW-1.5B 的平均 pass@1 分别下降 8.81 和 6.39 points。这个幅度远大于去掉 geometric normalization 的影响，说明 teacher signal 应该作用在哪些 trajectories 上，比单纯增加 teacher supervision 更重要。

这篇 paper 值得 post-training 团队认真看。它提供了一个清楚的设计原则：teacher preference 可以改善 RL 的 token-level learning signal，但需要受 advantage sign 约束。尤其在 cross-family distillation 中，selective guidance 比完整分布匹配更稳健。

今天第二篇是 **SWE-Pruner Pro: The Coder LLM Already Knows What to Prune**。

Coding agent 的长 trajectory 会不断累积文件内容、搜索结果和 tool output。已有方法通常训练一个额外 classifier 判断哪些代码行值得保留。SWE-Pruner Pro 发现，agent 自己读取 tool output 时形成的 hidden representation 已经包含 relevance signal。

作者给模型接入一个很小的 line-level keep-or-prune head，并加入与 tool output 行数相关的 length-aware embedding。Across two open-weight backbones and four multi-turn benchmarks，它最多节省 39% prompt 与 completion tokens，同时保持任务质量。在 MiMo-V2-Flash 上，SWE-bench Verified resolve rate 还提高了 3.8 points，Oolong accuracy 提高 2.2 points。

这项工作的价值很直接：context pruning 可以利用 agent 内部已经形成的 relevance representation。对于 open-weight coding agent，这可能比维护独立 classifier 更简单，也能把 token cost、latency 与 accuracy 放进同一个 learned component。

今天第三篇是 **TRIM: Reducing AI-Generated CodeSlop via Agent Trajectory Minimization**。

Coding agent 为了找到 passing solution，会留下 speculative edits、abandoned hypotheses 和 temporary scaffolding。测试通过后，这些残余修改仍可能留在最终 patch。作者把这种 functionally unnecessary residue 定义为 CodeSlop。

TRIM 利用完整 agent trajectory 识别并缩减这些残余。Across agentic scaffolds，它减少 17.9% 到 32.9% 的 CodeSlop，任务性能几乎不变，validation cost 约为 Delta Debugging baseline 的一半。

这个问题会随着 agent autonomy 扩大而累积。Solve rate 和 test pass rate 只能说明 patch 可用，无法覆盖后续维护成本。Trajectory 里恰好保存了搜索过程的 provenance，可以告诉 cleanup stage 哪些修改曾经服务于失败假设。

今天三篇 paper 给出三个很实用的信号：

1. teacher signal 需要按 trajectory outcome 选择性进入 RL update。
2. coding agent 的 internal state 可以直接支持 context pruning。
3. agent trajectory 同时也是 patch cleanup 的 provenance。

如果只读一篇，先读 `2607.17247`。Negative sample reset 的 ablation 很有说服力，而且实现思路足够简单，适合在现有 GRPO 或 on-policy distillation pipeline 里快速验证。

## 今日 3 篇精选

### 1) Distilled Reinforcement Learning for LLM Post-training
- 链接: https://arxiv.org/abs/2607.17247
- 摘要速读: 用 clipped reverse importance weights 把 teacher preference 写进 RL objective，并在 negative-advantage samples 上关闭 teacher reweighting。
- 为什么重要: 它在 within-family 和 cross-family distillation 中都超过 OPD、GRPO 与 OPD+RL，最关键的 negative reset ablation 带来 6.39 到 8.81 points 差异。

### 2) SWE-Pruner Pro: The Coder LLM Already Knows What to Prune
- 链接: https://arxiv.org/abs/2607.18213
- 摘要速读: 从 coder LLM 的 hidden representations 预测 tool-output 每一行的 relevance，最多节省 39% tokens。
- 为什么重要: learned context pruning 同时降低成本并保持任务质量，在一个 backbone 上还提升 SWE-bench Verified resolve rate。

### 3) TRIM: Reducing AI-Generated CodeSlop via Agent Trajectory Minimization
- 链接: https://arxiv.org/abs/2607.18161
- 摘要速读: 利用 agent trajectory 清理最终 patch 中遗留的 speculative 与 temporary edits。
- 为什么重要: 它把 maintainability 纳入 coding-agent evaluation，并以较低 validation cost 减少 17.9% 到 32.9% 的 CodeSlop。

## 一句话结论

今天最强的信号是：**dense teacher supervision 的价值取决于它进入哪些 RL trajectories，negative samples 上的 teacher guidance 可能直接破坏知识迁移。**
