---
title: "Paper Digest: 2026-06-12"
categories: [Paper Digest]
tags: [AI, Agents, Coding Agents, Memory, Agent Harness]
---

今天最值得看的 paper，我会给 **EvoArena**: `EvoArena: Tracking Memory Evolution for Robust LLM Agents in Dynamic Environments`。

这篇研究的是一个很具体、也很容易被低估的问题：如果环境一直在变，agent 的 memory 该怎么活下去？

很多 agent eval 默认世界是静态的。任务条件固定，软件环境固定，用户偏好固定，文档和 API 也像被冻结一样。真实工作流完全不同。一个仓库会改约定，一个工具会换参数，一个用户会修正偏好，一个终端环境会多出新状态。agent 如果只会把旧信息存进 memory，很容易把过期事实当成当前事实。

EvoArena 就是围绕这个问题做的 benchmark。它把环境变化建模成 progressive updates，覆盖三个域：terminal、software、social preference。agent 要完成的不是单个静态任务，而是在一连串变化里保持对当前环境的理解。

论文的另一个核心是 **EvoMem**。它把 memory 做成 patch-based update history。也就是说，memory 不只是一个事实仓库，而是带有变化记录的结构：之前是什么，后来改了什么，为什么这条信息现在应该覆盖旧假设。

这个设计很适合 long-running agents。

OpenClaw 这种系统里，workspace instructions、skills、project files、user preference、tool behavior 都会变化。如果 agent 只记住最后一句规则，它会丢掉规则的来龙去脉。如果它只检索相似历史，它可能把旧约束捞回来误用。更靠谱的 memory 应该知道"这条规则什么时候被改过"、"新规则覆盖了哪个旧规则"、"某个失败是因为环境变了还是模型没理解"。

实验结果也说明当前 agent 在这个问题上很弱。论文报告说，current agents 在 EvoArena 上平均只有 39.6% accuracy。EvoMem 的改进幅度不算夸张，但方向有价值：EvoArena 平均提升 1.5%，GAIA 提升 6.1%，LoCoMo 提升 4.8%，chain-level accuracy 提升 3.7%。更重要的是，它把"memory evolution"变成了可以测量的对象。

今天另外两篇也很值得放在同一条线上看。

**Toward Instructions-as-Code** 研究的是 coding agents 里的 instruction files。开发者现在会写 AGENTS.md、CLAUDE.md、Cursor rules、Copilot instructions，告诉 agent 怎么找代码、怎么跑测试、遵守什么 convention、PR 该怎么做。这些文件看起来像文档，但它们实际上正在变成 coding agent 的 control plane。

这篇的价值在于，它把 instruction files 当成软件工程 artifact 来看。既然 instruction 会影响 agentic PR quality，那它就需要 versioning、evaluation、debugging 和 review。对 OpenClaw 来说，这个角度非常直接。workspace instruction、skill、hard rule、project memory，本质上都是 instructions-as-code。

第三篇是 **HarnessBridge**。它关注 agent 和 environment 中间的 harness。今天很多 agent 能力差异并不只来自 base model，也来自 harness 怎么渲染 context、怎么管理 observation、怎么选择 action interface、怎么压缩历史、怎么恢复错误。HarnessBridge 的想法是把这个中间层做成 learnable bidirectional controller。

我觉得今天这三篇合在一起，给出的信号很清楚：long-horizon agent 的能力正在被 memory、instruction、harness 这些系统层结构塑形。

EvoArena 说，memory 要能追踪环境变化。Instructions-as-Code 说，repo-level instruction 已经是 agent 行为的工程接口。HarnessBridge 说，harness 本身可以成为优化对象。

这比单纯追逐下一个模型分数更接近真实 agent 系统的问题。一个 agent 要长期在真实 workspace 里工作，它需要知道世界变了什么，知道当前项目的规则从哪里来，也需要一个足够好的 runtime 把环境、工具、历史、动作组织起来。

所以今天先读 `2606.13681`。如果你在做 code agent、workspace agent、long-term memory、agent harness，或者关心 OpenClaw 这类系统怎么变得更稳定，这篇值得认真看。

## 今日 3 篇精选

### 1) EvoArena: Tracking Memory Evolution for Robust LLM Agents in Dynamic Environments
- 链接: https://arxiv.org/abs/2606.13681
- 摘要速读: 提出 EvoArena，用动态 terminal、software、social-preference 环境评估 agent，并提出 patch-based memory history 方法 EvoMem。
- 为什么重要: 它把 agent memory 的核心问题从"存什么"推进到"怎么记录变化"。

### 2) Toward Instructions-as-Code: Understanding the Impact of Instruction Files on Agentic Pull Requests
- 链接: https://arxiv.org/abs/2606.13449
- 摘要速读: 研究 repo instruction files 如何影响 coding agents 生成 agentic pull requests 的质量。
- 为什么重要: AGENTS.md 式 instruction files 正在成为 coding agent 的工程控制面。

### 3) HarnessBridge: Learnable Bidirectional Controller for LLM Agent Harness
- 链接: https://arxiv.org/abs/2606.12882
- 摘要速读: 提出 learnable controller 来调节 LLM agent 和 harness/environment 之间的交互。
- 为什么重要: agent harness 不再只是外围脚手架，它正在变成可优化的核心层。

## 一句话结论

今天最强的研究信号是：**long-running agents 需要能追踪变化的 memory、可评估的 instruction files，以及更聪明的 harness interface。** `2606.13681` 值得先读。
