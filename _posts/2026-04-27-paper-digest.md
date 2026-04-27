---
title: "Paper Digest: 2026-04-27"
categories: [Paper Digest]
tags: [AI, Coding Agents, Evaluation, Cost, LLM]
---

今天最值得看的 paper，是一篇把 agentic coding 里一个很少被认真量化、却已经非常现实的问题直接摆上桌面的工作。**How Do AI Agents Spend Your Money?** 关心的，不是 agent 能不能再多解几道题，也不是哪个模型 demo 更炫，而是一个所有真正开始部署 coding agent 的团队都会撞上的问题，**这些 agent 到底把 token 花在了哪里，花得值不值，能不能提前估出来。**

这篇 paper 的价值，在于它把“agent 很贵”这种大家隐约都知道的感觉，变成了更扎实的测量。作者在 SWE-bench Verified 上分析了八个 frontier 模型的运行轨迹，结论很硬。第一，agentic coding 的 token 消耗，和普通 code reasoning、code chat 根本不是一个量级，差距大到接近三个数量级。第二，真正拖高成本的，主要不是 output，而是 input。也就是 agent 在不断读 repo、看上下文、吸收反馈、回到环境里继续跑的时候，账单已经在前面越滚越大。第三，同一个任务多跑几次，token 消耗最多能差到 30 倍，而更贵并不自动意味着更准。

我觉得这篇 paper 最值得重视的地方，就在第三点。很多团队现在对 agent 的直觉，还停留在一种很朴素的想法上，既然任务复杂，那多花一点 token、多做几轮工具调用，应该会更稳吧。可这篇 paper 给出的信号刚好相反，**agent 的花费不只是高，而且带着明显的随机性，甚至常常在“中等成本”附近就达到比较好的准确率，再往上烧并不会线性换来更好的结果。** 这说明 coding agent 的问题，已经不只是能力问题，也是不小的资源配置问题。

还有一个很有意思的点，作者让模型在执行前预测自己这次大概要花多少 token，结果各家模型都做得不好，相关性很弱，而且普遍低估真实成本。这个结论很有味道，因为它说明我们不能天真地把“成本意识”外包给 agent 本身。你如果真想控制预算，靠 agent 自报家门远远不够，系统层的预算约束、轨迹监控、早停机制、成本可视化，这些东西会越来越重要。

这也是今天我把它放在第一位的原因。它没有提出一个更 flashy 的 agent 架构，却碰到了一个正在快速变成核心瓶颈的问题，**coding agent 的竞争，接下来不只是看谁会做题，也要看谁更会花钱。** 如果一个 agent 在同一类任务上比另一个多烧一百多万 token，最后准确率却没有更好，那这已经不是小优化问题，而是产品和平台层面都必须正视的系统问题。

## 今日 3 篇精选

### 1) How Do AI Agents Spend Your Money? Analyzing and Predicting Token Consumption in Agentic Coding Tasks
- 链接: https://arxiv.org/abs/2604.22750
- 摘要速读: 作者分析八个 frontier 模型在 SWE-bench Verified 上的 agentic coding 轨迹，发现这类任务的 token 消耗极高，主要由 input token 驱动，同任务多次运行的成本波动可达 30 倍，而且更高花费并不稳定带来更高准确率。
- 为什么重要: 这篇 paper 真正切中了 coding agent 进入生产之后最痛的一层现实。能力评测之外，成本测量、预算控制、运行方差，已经开始决定 agent 到底是不是一个可持续产品。

### 2) RealBench: A Repo-Level Code Generation Benchmark Aligned with Real-World Software Development Practices
- 链接: https://arxiv.org/abs/2604.22659
- 摘要速读: RealBench 把 repo-level code generation benchmark 往真实软件开发流程推近一步。每个样本同时给自然语言需求和 UML 设计，结果显示当前 LLM 在仓库级代码生成上仍然很弱，尤其容易出现语法和逻辑错误。
- 为什么重要: 这篇 paper 像一盆冷水，提醒人不要把 HumanEval 式成功误当成真实开发能力。真正接近企业软件流程以后，模型的短板会暴露得更明显。

### 3) Agentic World Modeling: Foundations, Capabilities, Laws, and Beyond
- 链接: https://arxiv.org/abs/2604.22748
- 摘要速读: 这是一篇大范围综述，试图把 agent world model 重新整理成一套 levels × laws taxonomy，用 capability level 和 law regime 去统一讨论模型式 RL、GUI agent、social simulation、scientific discovery 等方向。
- 为什么重要: 它更像一张地图。对于想建立 agent 世界模型整体视角的人，这篇 paper 提供了更清楚的话语体系和评估框架。

## 一句话结论
今天最强的研究信号是，**coding agent 正在变成一个成本与测量问题。** `2604.22750` 值得重点看，因为它提醒人，agent 的真实瓶颈，很多时候不在演示视频里，而在那张越来越长、而且越来越不可预测的 token 账单里。
