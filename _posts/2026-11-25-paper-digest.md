---
title: "Paper Digest: 2026-11-25"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, GRPO, Post-Training]
---

今天最值得看的 paper 是 **Lazy Agents Answer Against Their Own Beliefs: Reinforcement Learning with Verifiable Beliefs Teaches Them to Act on What They Believe**。

长任务里的 agent 有一种常见失败：证据还不够，它已经提交答案。更微妙的是，agent 自己写出的 belief 可能仍然列着多个候选，但 action 已经选择 commit。论文把这种行为称为 laziness。

常见 outcome reward 只检查最终答案。一次缺乏证据的幸运猜测和一次扎实完成信息收集后的正确回答，会得到完全相同的 reward。这样的信号无法教会 agent 何时继续探索、何时停止。

作者提出 Reinforcement Learning with Verifiable Beliefs，简称 RLVB。核心思路很直接：训练环境能够计算每一步的真实 posterior，就用它判断当前证据是否已经足够。多个答案仍然可能时，奖励继续行动；posterior 已经收敛时，奖励提交答案。

## Agent 的问题经常出在 commit timing

长任务策略至少包含两类决策：下一步做什么，以及什么时候结束。Terminal reward 能评价最终结果，却把整个轨迹压缩成一个标量。只要答案碰巧正确，过早结束也会获得正反馈。

这会产生一个危险的训练结果。Policy 学到更强的猜测倾向，短期 success rate 可能上升，证据驱动的行为却在退化。论文实验里，使用 outcome reward 的 GRPO 和 PPO 平均表现都低于 base model，正好说明“可验证的最终答案”仍可能是一个不完整的训练目标。

RLVB 使用训练环境的 privileged state 补上时间维度。它不读取 agent 自己写出的 belief line，也不需要 LLM judge、critic 或 learned reward model。Simulator 直接计算答案 posterior，并判断当前观察是否还能支持多个答案。

这个 reward 回答两个问题：

- 当前不确定性仍高时，agent 有没有继续收集信息？
- 证据已经充分时，agent 有没有及时 commit？

部署时 agent 不需要访问 posterior。Oracle 只存在于训练环境中，用来塑造策略。

## 结果比 outcome-only RL 更稳定

作者同时发布 Pro Worlds，一个视觉谜题 benchmark。Agent 只能通过小窗口观察大型世界，需要多步移动视野、积累证据，最后回答问题。

实验覆盖三个 vision-language models 和三个 benchmarks，其中两个 benchmark 没有出现在训练中。RLVB 在全部九个设置里都提升 base model，平均增益为 17.1%。

相比使用 LLM judge process reward 的 PPO，RLVB 平均高 9.3%，GPU-hours 还少 1.6 倍。训练后的 agent 在三个 benchmark 上以充分证据作答的比例分别达到 51%、12% 和 74%，base model 对应为 8%、0.4% 和 36%。

作者还检查了 belief line 与 action 的一致性。证据不足且 belief 仍保留多个答案时，RLVB agent 的 commit 比例只有 0.1% 到 2.2%，base model 则为 3% 到 68%。性能提升确实来自更好的信息收集和停止时机。

## 对 agent post-training 的启发

RLVB 最值得借鉴的部分是 reward interface 的设计。很多 agent 任务拥有训练期可见、部署期不可见的 privileged state，例如：

- 搜索环境中的完整文档集合和答案 provenance
- 软件工程环境中的 reference behavior、隐藏测试覆盖和执行状态
- Computer-use simulator 中的真实 UI state 与目标完成条件
- 规划环境中的完整 world state 和剩余可行解集合

这些信息可以构造便宜、确定、可复现的 process signal，监督信息收集、停止条件和 action timing。它们不必进入部署 prompt，也不需要增加在线 inference 成本。

对 GRPO pipeline 来说，这篇论文也提供了一个明确检查项：terminal reward 是否能区分正确答案的获得过程？如果 lucky guess 与 evidence-backed solution 得分相同，继续扩大 rollout 数量可能会强化错误的时间策略。

真正困难的部分是把方法带到开放世界。Pro Worlds 中可以精确计算 posterior，真实 coding 或 browsing 任务很少拥有同样干净的概率分布。更现实的落地路径可能是先构造可完全验证的 synthetic curriculum，让 agent 学会探索和 commit 的节奏，再迁移到开放任务。

## 今天另外两篇相关论文

**RECAP: Training Long-horizon LLM Agents with Context Compaction** 把 context compaction 定义成 behavior preservation。它比较完整 history 和 summary 驱动的后续 action distribution，用 behavior distortion 同时训练 summary generator 和 compaction 后的 policy。在 Qwen3.5-4B 上，它相对 task-reward-only RL 在 BrowseComp-plus 和 SWE-bench Verified 分别提升 4.6 和 4.3 个点。这对长 coding session 很直接：摘要看起来完整还不够，它必须保留会改变后续工具调用和代码编辑的状态。

**InstructReplay** 研究 factual fine-tuning 导致的能力退化。标准 fine-tuning 让 instruction following、reasoning 和 coding benchmarks 平均下降 35 分，同时回答长度至少缩短一半。它先让初始模型回答一组通用 instructions，再在后续 fine-tuning 中 replay 这些 assistant turns，把五项通用 benchmark 的平均退化从 28.1 分降到 2.4 分，同时保持新知识学习效果。

论文：<https://arxiv.org/abs/2611.32884>

另外两篇：<https://arxiv.org/abs/2611.31258>、<https://arxiv.org/abs/2611.32918>
