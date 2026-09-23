---
title: "Paper Digest: 2026-09-23"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Code Agents, Post-training, Distillation]
---

今天最值得看的 paper 是 **The Tasteful Agent: Measuring and Improving Taste in Long-Horizon Tasks**。

一个 coding agent 可以在前几步做出完全合理的选择，跑了很久才发现方向错了。最终 success rate 会记录失败，却很难告诉我们真正关键的错误发生在哪里。论文把这种能力称为 agent 的 taste：在结果尚未出现时，能否选中更值得继续投入的方向。

作者从真实的 software engineering 和 ML research trajectories 中提取了 502 个 decision forks。当前最强模型也只答对 59.7%。当判断依据需要更久以后才出现，所有模型都会明显变差；增加 reasoning budget 没有改善这个问题。

更重要的是，taste 可以训练。作者让知道最终结果的 teacher 生成完整判断过程，再把这份 privileged hindsight 蒸馏给只能看到岔路现场的 student。Student 在未见任务上的判断准确率提升 17.9 个百分点。把它作为 advisor 接入固定 executor 后，41 个 held-out SWE-bench Pro tasks 的成功率从 14.6% 提高到 33.7%。

## 什么是 decision fork

Decision fork 是 trajectory 中一个足以改变后续结果的岔路点。模型在这里面对两个当下都说得通的方向，例如：

- 保留当前模型规模，缩短训练步数
- 缩小模型，换取更多训练步数

只有看到后续 loss、测试结果或任务是否完成，才能判断哪条路更好。评测时，这些未来证据会被隐藏，模型只能看到 task、fork 之前的 trajectory prefix 和两个候选方向。

作者用两种方式挖掘 forks。

第一种来自 parallel trajectories。多个 agent 尝试共享近似的前缀，随后选择不同方向，最终 outcome 为岔路提供标签。

第二种来自 detour trajectory。Agent 先走了一条路，观察到失败，再回头选择另一条路并完成任务。被放弃的方向和后来有效的修正构成一组比较。

这种数据很适合 agent post-training。它来自已经支付过推理成本的 rollouts，团队无需再让专家逐步标注。一次 rollout 除了贡献 pass/fail，还能产出多组 process-level supervision。

## Taste-Bench 暴露了什么

Taste-Bench 覆盖 software engineering 与 ML research。14 个 frontier models 中，GPT-5.6 Sol 最高也只有 59.7%。

论文进一步标注了每个 fork 的 time horizon，也就是需要向未来看多远，才能获得支持正确选择的证据。Horizon 越长，准确率越低。两个模型使用更高 reasoning effort 重跑后，结果几乎没有变化。

这说明当前模型遇到的瓶颈无法单靠更多 test-time thinking 化解。它们缺少对后续工作结构的可靠预测，尤其难以判断某个局部选择会怎样影响几十步之后的修改、验证与恢复成本。

Taste-Bench 与 SWE-bench Verified 的 Pearson correlation 是 0.63，在 engineering subset 上只有 0.37。几个 SWE-bench 分数接近的强模型，在 Taste-Bench 上能相差 10.7 个百分点。最终解题能力与途中判断质量有联系，但两者并不等价。

## 用 hindsight 蒸馏判断力

训练阶段使用 Qwen3.6-27B，并更新 LoRA adapters。Teacher 和 student 来自同一个 frozen base model，区别在于上下文：teacher 能看到 supported candidate 的提示，student 只能看到正常评测信息。

Teacher 生成包含完整 reasoning 和最终选择的 continuation。训练用 token-level forward KL，把这些 reasoning tokens 与 choice 蒸馏给 student。工程题按照 source task 划分成两个 task-disjoint folds，避免同一任务同时出现在训练与评测中。

结果很清楚：在 held-out engineering questions 上，student accuracy 从 30.0% 提升到 47.9%，增加 17.9 points。按两个选项顺序的平均准确率计算，则从 42.7% 提升到 62.4%。

End-to-end 测试中，固定 Qwen3.6-27B executor 在没有 advice 时成功率为 14.6%。如果每个 fork 都提供正确 advice，上限达到 39.0%；使用 student advice 时达到 33.7%，获得其中大部分收益。

## 对 code agent 训练系统的启发

最直接的做法是给 rollout logging 增加几类结构化记录：

- 可对齐的 trajectory prefix
- 当时考虑过的 candidate plans
- 后续测试、reward 与 verification evidence
- 被放弃的 detour 及恢复动作
- fork 到决定性证据之间的 time horizon

这些记录可以生成 pairwise preference data、advisor SFT data，或 process reward model 的训练样本。评测也应按 time horizon 分桶。短期选择做得很好，可能掩盖 agent 在长周期 planning 上的薄弱判断。

需要谨慎看待标签。两条 branch 的最终差异仍可能混入 execution quality 和 environment randomness。作者通过共享前缀、过滤歧义问题与 human review 控制噪声，生产系统还需要多次 rollout、因果归因和置信度门槛。

另一个限制是 end-to-end advice 来自这些 held-out tasks 的早期 runs，虽然 student 在训练时没见过这些任务，但 pipeline 已经为同一任务挖出了 forks。它验证了判断蒸馏能够改善执行，距离对全新任务在线发现关键 fork 还有一步。

## 今天另外两篇相关论文

**CliffCompaction** 研究 long-horizon coding agent 的 context 压缩。它只保留、截断或删除原始内容，拒绝改写，并且每次都从 canonical original history 重新生成 compact view，避免 summary of summary 带来的 drift。该方法把 bounded-context cost 最多降低 50%，还支持超过一百万 tokens 的持续 session。

**Recursive self-improvement of AI research agents** 提出 AIDE^2，让 research agent 修改自己的代码、跑 benchmark、通过 hidden evaluation 保留更好的版本。八天 autonomous run 累积了七次改进，最终 agent 在四个 held-out benchmarks 上达到或超过强人工系统，同时 reward hacking rate 从 55% 降至 32%。

三篇论文共同指向一个很具体的工程方向：保留完整 trajectory provenance，把失败、分岔、context 管理和 agent patch 都变成可验证的数据。Long-horizon agent 的下一批训练信号，很可能已经躺在现有 rollout logs 里。

论文：<https://arxiv.org/abs/2609.25804>

另外两篇：<https://arxiv.org/abs/2609.26779>、<https://arxiv.org/abs/2609.26457>
