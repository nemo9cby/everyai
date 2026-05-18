---
title: "Paper Digest: 2026-05-18"
categories: [Paper Digest]
tags: [AI, Agents, Coding Agents, Software Engineering, Post-Training]
---

今天最值得看的 paper，我会给 **RoadmapBench**。

这篇 paper 的价值很直接，它终于把“coding agent 看起来会修 bug”和“coding agent 真的能做版本级开发”分开测了。

RoadmapBench 做的是 version upgrade 任务。agent 拿到的是旧版本代码快照，目标不是修一个孤立 issue，而是根据 roadmap instruction，把系统推进到更接近新版本的状态。这里面往往不是一两个文件的修改，而是跨模块、跨接口、跨行为的一整串联动。

这个 benchmark 最有分量的地方，是它的任务规模很像真实工程。115 个任务，来自 17 个 repository、5 种编程语言，中位数改动规模是 3700 行、51 个文件。这个量级一出来，很多过去在 bug-fix benchmark 上还算体面的成绩，立刻显得乐观了。

结果也很诚实。13 个 frontier model 里，最强的 Claude-Opus-4.7 也只解决了 39.1% 的任务，最弱的只有 5.2%。这组数字很重要，因为它提醒我们，今天的 agent 还远没有穿过真正的软件工程门槛。

我觉得这篇 paper 值得优先读，不只是因为它把分数打低了，而是因为它把问题打准了。

现在很多 coding agent 评测，还是围绕单点 bug fix、单函数编辑、或者局部测试通过率展开。这些任务当然有价值，但它们太容易让人误判系统能力。真实开发更常见的难点，是长期依赖、上下游兼容、接口迁移、多文件协同，还有中间状态怎么维持。RoadmapBench 把这些东西一起拉进来了。

这会逼着 agent 系统重新面对几个更硬的问题：

- planning 怎么从单轮决策变成阶段化 roadmap execution
- context management 怎么覆盖几十个文件的持续状态
- verification 怎么从单点 test pass 变成多目标验收
- benchmark 设计怎么更接近 repo-level engineering reality

今天另外两篇也值得一起看。

**SkillSmith** 很像 runtime architecture 方向的一记重锤。它的核心观点是，skill 不该永远以大段上下文形式塞回 prompt。很多 skill 可以先离线编译成 boundary-guided interface，让 agent 在运行时只调用真正相关的部分。这个设计在 token、时间、thinking iteration 上都带来了很扎实的下降。

**Learning to Foresee** 则是 post-training 视角里更偏机制的一篇。它试图解释 on-policy distillation 为什么高效，重点放在 update trajectory 的早期稳定性和 module allocation 上。对做 reasoning / RL / distillation 的团队来说，这种机制解释比单纯报分更有后续价值。

如果把今天这三篇放在一起看，我觉得能看到一个挺清楚的信号：

真正限制 agent 前进速度的，越来越不是一句“模型更强就行”，而是工程结构本身。评测要更像真实开发，skill 接口要更像运行时系统，训练配方也要更懂参数到底在往哪里走。

所以今天最推荐先读 `2605.15846`。它不是最花哨的一篇，但它把现实摆得很平，agent 距离真正的软件开发，还有很长一段路要走。

## 今日 3 篇精选

### 1) RoadmapBench: Evaluating Long-Horizon Agentic Software Development Across Version Upgrades
- 链接: https://arxiv.org/abs/2605.15846
- 摘要速读: 用真实开源项目版本升级构造 115 个长时程 coding 任务，中位数 3700 行改动、51 个文件，最强模型也只做对 39.1%。
- 为什么重要: 它把 coding agent 拉回真实软件工程尺度，测的不是单点修补，而是多目标、多文件、长链路开发。

### 2) SkillSmith: Compiling Agent Skills into Boundary-Guided Runtime Interfaces
- 链接: https://arxiv.org/abs/2605.15215
- 摘要速读: 把 agent skill 离线编译成最小可执行接口，减少无关上下文注入和重复 reasoning。
- 为什么重要: 它说明 skill 设计开始从 prompt engineering 走向 runtime systems engineering。

### 3) Learning to Foresee: Unveiling the Unlocking Efficiency of On-Policy Distillation
- 链接: https://arxiv.org/abs/2605.11739
- 摘要速读: 从参数更新轨迹角度解释 OPD 的高效性，并据此提出更省的 EffOPD recipe。
- 为什么重要: 它给 post-training 提供了更机制化的理解，而不只是一个“好用配方”。

## 一句话结论
今天最强的研究信号是，**agent 的上限越来越取决于系统结构本身。** RoadmapBench 在评测层面把现实压得更重，SkillSmith 在运行时层面把 skill 变得更像基础设施，Learning to Foresee 则在训练层面解释效率从哪里来。`2605.15846` 值得先读。