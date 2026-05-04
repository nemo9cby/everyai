---
title: "Paper Digest: 2026-05-04"
categories: [Paper Digest]
tags: [AI, Agents, Code, Post-Training]
---

今天最值得看的 paper，我会给 **Can Coding Agents Reproduce Findings in Computational Materials Science?**。

这篇 paper 的价值，不在于又做了一个“agent benchmark”，而在于它故意把 coding agent 从熟悉的 SWE 题库里拎出来，丢进一个更麻烦、也更真实的场景：**你能不能根据论文里并不完整的描述，复现实验流程，并判断结果到底支不支持原来的 scientific claim**。

作者做了一个 benchmark，叫 **AutoMat**。它针对 computational materials science 里的真实论文 claim，要求 agent 完成三件事：先补全 paper 里没写清楚的 procedure，再跑通专门的 scientific toolchain，最后还要解释实验结果到底算不算支持这个 claim。这里最难的地方，不只是“写代码”，而是要在含糊描述、复杂依赖和 scientific judgment 之间来回切换。

结果也很说明问题。作者测试了几种代表性的 coding-agent setup，最好的 success rate 也只有 **54.1%**。这说明今天很多 coding agent 的强，仍然很大程度建立在 benchmark task 比较干净、目标比较明确、执行路径比较短的前提上。一旦进入真实科研 workflow，失败模式马上就冒出来了：步骤补不全、方法跑偏、执行脆弱，最后连 claim 是否成立都判断不稳。

我觉得这篇 paper 最值得注意的地方，是它把 agent 能力评估从“会不会生成代码”推进到“能不能恢复一条真实工作链路”。这个变化很关键。因为很多高价值任务，本质上都不是一段 isolated code completion，而是带着上下文缺口、工具摩擦和结果解释的长流程任务。你如果只在传统 SWE benchmark 上看分数，很容易高估 agent 的泛化能力。

对做 agent 的人来说，这篇 paper 还有一个更现实的提醒：**workflow reconstruction 本身就是能力边界。** 用户给你的永远不是完整 spec，尤其在 research、ops、analysis 这些场景里，真正的难点常常是把零散描述拼成可执行程序，而不是补一小段函数。AutoMat 抓到的就是这个层面。

如果要说今天第二重要的 paper，我会看 **Themis**。它讲的是 code reward model 不应该只盯 functional correctness，而应该学会多维度、多语言地打分。这对代码 post-training 很重要。第三篇 **RECRL** 也值得留意，它把 requirement difficulty 引进 curriculum RL，用 requirements engineering 的视角来改 code generation 训练。

## 今日 3 篇精选

### 1) Can Coding Agents Reproduce Findings in Computational Materials Science?
- 链接: https://arxiv.org/abs/2605.00803
- 摘要速读: 提出 AutoMat benchmark，测试 coding agent 能否从真实材料科学论文里恢复流程、调用专门 toolchain，并判断结果是否支撑 scientific claim。
- 为什么重要: 它说明 agent 一旦离开熟悉的 SWE benchmark，进入真实 scientific workflow，鲁棒性会明显下降。

### 2) Themis: Training Robust Multilingual Code Reward Models for Flexible Multi-Criteria Scoring
- 链接: https://arxiv.org/abs/2605.00754
- 摘要速读: 构建多语言、多维度 code reward benchmark 和 35 万+ preference pairs，训练从 600M 到 32B 的 code reward models。
- 为什么重要: 这篇把 code post-training 的 reward infrastructure 往前推了一步，不再只看 execution feedback。

### 3) Improving LLM Code Generation via Requirement-Aware Curriculum Reinforcement Learning
- 链接: https://arxiv.org/abs/2605.00433
- 摘要速读: 提出 RECRL，根据 requirement 难度来组织 code RL 课程，并优化高难 requirement 的训练利用率。
- 为什么重要: 它提醒我们 code generation 的 curriculum，不该只围着答案转，而要认真处理 requirement 本身。

## 一句话结论
今天最强的研究信号是，**代码智能的下一步，不只是更会写代码，而是更会被评测、更会被奖励，也更能在不完整真实流程里活下来。** `2605.00803` 值得优先读，因为它把 coding agent 的真实短板暴露得很干脆。
