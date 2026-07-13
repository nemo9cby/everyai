---
title: "Paper Digest: 2026-07-13"
categories: [Paper Digest]
tags: [AI, Coding Agents, LLM Agents, Evaluation, Software Engineering, Post-training]
---

今天最值得看的 paper，我会选 **Long-Horizon-Terminal-Bench: Testing the Limits of Agents on Long-Horizon Terminal Tasks with Dense Reward-Based Grading**。

这篇 paper 很直接地戳中了当前 agent 的薄弱处：短任务看起来越来越能做，长任务还是很难。

很多 terminal agent benchmark 最后只看一个结果：任务完成了吗？测试过了吗？但真实的长流程任务很少这么简单。agent 可能前半段做对了，后半段迷路了；也可能已经完成了 70% 的关键步骤，只是在最后一个环境问题上失败。只看最终结果，会把这些差异全部压成一个 bit。

Long-Horizon-Terminal-Bench 的设计重点是 **dense reward-based grading**。

它把 46 个 terminal tasks 拆成更细的 graded subtasks，覆盖 experiment reproduction、software engineering、multimodal analysis、interactive games、scientific computing 等 9 类任务。这样一来，评测可以看到 agent 到底推进到了哪一步，而不是只问它最后有没有撞线。

论文里的数字很醒目。

在这些任务上，agents 平均每个任务消耗 **9.9M tokens**，经历大约 **231 episodes**，运行 **85.3 minutes**。这已经不是普通问答或短代码修复的规模，更像一个小型工程任务。

结果也说明差距还很大。

最强模型在 partial-reward threshold 0.95 下的 pass@1 只有 **15.2%**，完美 reward threshold 1.0 下只有 **10.9%**。所有模型平均只有 **4.3%** 和 **1.7%**。

我觉得这篇 paper 的价值在两个地方。

第一，它把 long-horizon agent 的难度量化了。

当一个任务需要上百轮尝试、上千万 token、一个多小时运行时间时，问题就不只是模型会不会写下一步命令。真正的难点包括规划、上下文管理、状态恢复、错误诊断、长程验证，以及中间成果不丢失。

第二，它给 post-training 和 eval 都留下了接口。

如果评测能给 dense intermediate rewards，未来训练 coding agent 或 terminal agent 时，就可以利用更细的监督信号。final pass/fail 太稀疏，尤其对长轨迹 RL 来说很难学。LHTB 至少把这个问题具体化了。

今天第二篇 **Failure as a Process: An Anatomy of CLI Coding Agent Trajectories** 则从另一个角度看同一件事。

它研究 CLI coding agent 的失败过程，而不是只看失败结果。

作者收集了 3,843 条 execution trajectories，来自 7 个 frontier models、3 个 coding-agent scaffolds（OpenHands、MiniSWE、Terminus2），任务来自 Terminal-Bench。最后他们人工标注了 1,794 条完整有效轨迹，覆盖超过 63,000 个 execution steps。

结论挺实用：很多失败在最开始几步就已经埋下了。

agent 可能一开始就误读需求、选错文件、做出错误假设，但这些错误不会马上暴露。它会继续执行，继续生成看似合理的步骤，直到后面已经很难恢复。

这对 agent 产品很重要。

如果失败经常早早开始，harness 就不能只在最后跑测试。它应该更早检查关键假设：agent 有没有读对 issue？有没有定位到正确模块？有没有验证环境？有没有把中间观察写回状态？

第三篇 **TTHE: Test-Time Harness Evolution** 更像一个可能的解决方向。

它的核心观点是：agent 的能力不只来自 base model，也来自外面的 harness。

harness 负责构造上下文、调用工具、验证中间结果、处理失败恢复。过去很多方法是在部署前搜索一个固定 workflow，然后测试时冻结它。TTHE 问的是：能不能在测试过程中，根据 unlabeled execution traces 继续改进 harness？

它不更新模型权重，也不需要 gold labels。它维护一组 candidate harnesses，让 proposer 根据 execution traces 提出改进，再用 proxy signals 选择新的 harness，让这个改进持续影响后续输入。

这个方向很适合真实 agent 系统。

因为生产中的 agent 往往不是缺一个更长的 prompt，而是缺更好的控制程序：什么时候检索，什么时候验证，什么时候重试，什么时候停止，什么时候把失败状态保存下来。TTHE 把这些控制逻辑当成可演化的程序，而不是一次性 prompt。

今天这三篇放在一起，主题很集中：

**long-horizon agents 需要围绕 trajectory 重新设计。**

LHTB 解决怎么给长任务更细的分数。Failure as a Process 解释失败怎样在轨迹里发生。TTHE 尝试让 harness 根据轨迹持续进化。

如果你关心 coding agents、terminal agents、agentic RL、post-training、agent eval，今天先读 `2607.08964`。

## 今日 3 篇精选

### 1) Long-Horizon-Terminal-Bench: Testing the Limits of Agents on Long-Horizon Terminal Tasks with Dense Reward-Based Grading
- 链接: https://arxiv.org/abs/2607.08964
- 摘要速读: 构造 46 个 long-horizon terminal tasks，用 dense reward-based grading 给 agent 的中间进展打分，而不只看最终 pass/fail。
- 为什么重要: 当前最强模型在这些任务上仍然只有 15.2% partial-threshold pass@1 和 10.9% perfect-threshold pass@1，长轨迹 agent 还有大量空间。

### 2) Failure as a Process: An Anatomy of CLI Coding Agent Trajectories
- 链接: https://arxiv.org/abs/2607.09510
- 摘要速读: 人工分析 1,794 条 CLI coding-agent trajectories 和 63,000 多个 execution steps，研究失败从哪里开始、如何演化、何时变得不可恢复。
- 为什么重要: 很多失败在最早几步就出现，但直到后面才暴露。agent harness 需要更早做假设检查和恢复干预。

### 3) TTHE: Test-Time Harness Evolution
- 链接: https://arxiv.org/abs/2607.08124
- 摘要速读: 在测试时用 unlabeled execution traces 演化 agent harness，不改模型权重，通过改进工具调用、验证和恢复逻辑提升 agent 表现。
- 为什么重要: 真实 agent 的进步很可能来自可执行 harness 的持续优化，而不只是换更大的 base model。

## 一句话结论

今天最强的新信号是：**长轨迹 agent 的核心问题要放在 trajectory 里看，评测、诊断、训练和 harness 都要围绕中间过程设计。** `2607.08964` 值得先读。
