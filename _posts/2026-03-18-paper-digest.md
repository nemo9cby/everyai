---
title: "Paper Digest: 2026-03-18"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天 papers 这条线最清楚的主线，是 agent 和 code model 正在从“会不会做 demo”转向“能不能在真实约束里稳定工作”。一个是工业代码能力开始有了更像样的训练 recipe，一个是 deep research agent 开始把 verification 内置到 reasoning loop，另一个是 repo-level 理解终于被当成独立能力来认真评测。

## 今日 3 篇精选

### 1) InCoder-32B: Code Foundation Model for Industrial Scenarios
- 链接: https://arxiv.org/abs/2603.16790
- 摘要速读: InCoder-32B 想解决的不是通用 LeetCode 式代码能力，而是工业环境里真正难的部分，比如 chip design、GPU kernel optimization、embedded systems、compiler optimization、3D modeling。它的 recipe 也很明确，先做 general code pretraining，再做工业代码 annealing，然后把 context 从 8K 拉到 128K，并加入 synthetic industrial reasoning data，最后再用 execution-grounded verification 做 post-training。
- 为什么重要: 这篇的价值在于它把“工业代码能力”从一句口号拆成了数据、上下文和验证三层工程问题。真正值得盯的，不是某个 benchmark 点数，而是这套 recipe 会不会成为后续工业 code model 的主流路线。

### 2) MiroThinker-1.7 & H1: Towards Heavy-Duty Research Agents via Verification
- 链接: https://arxiv.org/abs/2603.15726
- 摘要速读: MiroThinker-1.7 强化 structured planning、contextual reasoning 和 tool interaction，H1 则进一步把 verification 直接塞进推理过程，让中间决策和整体轨迹都能被检查。重点不在“更长思维链”，而在“更可审计的长程推理”。
- 为什么重要: 现在很多 research agent 看起来会做复杂任务，但问题常出在中间步骤不可控、证据链不干净。这篇值得看，因为它把 verification 从一个外部评测动作，变成了 agent 推理架构的一部分。

### 3) SWE-QA-Pro: A Representative Benchmark and Scalable Training Recipe for Repository-Level Code Understanding
- 链接: https://arxiv.org/abs/2603.16124
- 摘要速读: SWE-QA-Pro 做 repo-level 代码理解 benchmark，刻意避开那些模型可能已经记熟的热门仓库，转向 long-tail repo 和可执行环境。它想测的不是“模型会不会回答一个代码问题”，而是“模型能不能在陌生代码库里建立正确的 mental map”。
- 为什么重要: 很多 SWE agent 失败，表面像是不会修 bug，底层其实是 repo 理解没打牢。只要这个能力层还薄弱，后面的规划、修改、验证都会发虚。

## 可能感兴趣

### 4) TRACE: Evaluating Execution Efficiency of LLM-Based Code Translation
- 链接: https://arxiv.org/abs/2603.16479
- 亮点: 它提醒大家，code translation 不该只测 functional correctness，还得测执行效率。能跑通但慢很多，在真实工程里照样可能是坏结果。

## 一句话结论
今天最值得记住的是，代码与 agent 方向的竞争，正在从“谁更会写”转向“谁更会在真实约束里被训练、被验证、被理解”。这条线比单次 benchmark 胜负更重要，也更接近真正的产品上限。
