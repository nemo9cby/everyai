---
title: "Paper Digest: 2026-03-30"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天最值得记住的研究信号，不在模型又多会答题一点，而在 agent 系统终于更认真地面对一个更难的问题：**经验怎么沉淀**。一条轨迹能跑通，不代表能力真的留下来了。真正有价值的，是把一次次执行里的局部教训，压缩成以后还能复用的 skill、memory 和 repo-specific know-how。

## 今日 3 篇精选

### 1) Trace2Skill: Distill Trajectory-Local Lessons into Transferable Agent Skills
- 链接: https://arxiv.org/abs/2603.25158
- 摘要速读: 这篇 paper 瞄准了 agent 一个很核心、也很容易被忽略的问题。我们手里越来越多 execution traces，但这些轨迹大多是一次性消费品，跑完就散了。Trace2Skill 想做的是，把分散在不同轨迹里的局部经验收拢起来。它不是顺着单条轨迹一路总结，而是并行派出 sub-agents 去分析一批执行过程，先提取 trajectory-local lessons，再往上归纳成统一、无冲突、可迁移的 skill directory。
- 为什么重要: 这篇的价值，在于它把“agent 变强”从 prompt 调参和模型升级，往 knowledge packaging 推了一步。很多系统的问题，不是没有经验，而是经验沉不下来。只要 skill 还是碎片化的，agent 就会重复犯错、重复探索。Trace2Skill 这条路，如果能走通，会让 agent improvement 更像工程积累，而不是一次次碰运气。

### 2) Learning to Commit: Generating Organic Pull Requests via Online Repository Memory
- 链接: https://arxiv.org/abs/2603.26664
- 摘要速读: 这篇来自 cs.SE，切中 code agent 很真实的痛点。很多 agent 生成的 PR 功能上未必错，但 maintainer 一眼就会觉得“不像这个 repo 里长出来的东西”。作者把这个问题叫 organicity，然后提出 Online Repository Memory。做法是按时间顺序回放历史 issue 和 commit，让 agent 先盲做，再和真实 diff 对比，把风格、内部 API 用法、架构约束这些模式提炼成可复用记忆。
- 为什么重要: 现在 code agent benchmark 很多，但“真实 repo 会不会接受这份改动”常常没被认真当成主指标。这篇 paper 迷人的地方，在于它不满足于 patch 能跑，而是追问 patch 是否属于这个项目。这个判断比 pass 一个 toy task 更接近生产环境。

### 3) Natural-Language Agent Harnesses
- 链接: https://arxiv.org/abs/2603.25723
- 摘要速读: 这篇讨论的是另一个经常被埋在 glue code 里的层，harness engineering。作者提出 Natural-Language Agent Harnesses，把 agent 的高层控制逻辑写成可编辑的自然语言 artifact，再配一个共享 runtime 去执行。换句话说，它想把“控制器到底怎么 orchestrate agent”从隐蔽代码里抽出来，变成可以迁移、比较、研究的对象。
- 为什么重要: 这类工作还谈不上已经证明新范式成立，但方向很值得看。越来越多 agent 系统的差异，不只在 base model，而在 harness 层。谁把这个层次讲清楚，谁就更有机会把 agent 从 demo 堆栈带到可复用系统。

## 可能感兴趣

### 4) RealChart2Code: Advancing Chart-to-Code Generation with Real Data and Multi-Task Evaluation
- 链接: https://arxiv.org/abs/2603.25804
- 亮点: 用真实数据和多轮 refinement 去测 chart-to-code generation，说明 VLM 在复杂可视化代码生成上离“真的会做”还有明显距离。

## 一句话结论
今天最强的主线是，agent research 开始从“会不会完成任务”走向“经验能不能留下来”。无论是 Trace2Skill 的可迁移 skill，还是 Learning to Commit 的 repository memory，背后都在回答同一个问题：下一代 agent 的护城河，也许不是更长的推理链，而是更会把过去变成以后。