---
title: "Paper Digest: 2026-05-06"
categories: [Paper Digest]
tags: [AI, Code, Agents, Software-Engineering]
---

今天最值得看的 paper，我会给 **ProgramBench: Can Language Models Rebuild Programs From Scratch?**。

这篇 paper 很直接，它问的就是一句很多人嘴上已经在默认、但证据其实并不够的话：**LLM agent 到底能不能从零搭起一个像样的软件项目？**

作者做法很硬。他们没有继续考单点 bug fix、局部 feature completion 这种相对容易的题，而是提出 ProgramBench，让模型面对 200 个完整软件任务。输入只有一个 reference executable 和相关文档，agent 需要自己做架构决策、写代码、把整个项目搭出来，最后再用 agent-driven fuzzing 生成的 end-to-end behavioral tests 去验收。

结果很说明问题。9 个模型里，没有一个能完整解决任何任务。表现最好的模型，也只是在 3% 的任务上通过了 95% 的测试。更扎眼的是，模型普遍倾向于写成 monolithic、single-file implementations，和人类写出的软件结构差得很远。

我觉得这篇最有价值的地方，在于它把 code agent 的讨论从 demo 感很强的局部成功，拉回到真正的软件工程语境里。做一个能跑的函数，和搭一个能长期生长的代码库，中间差的不只是 token 和工具调用次数，中间差的是 architecture judgment、模块划分、约束管理、以及一连串长期决策的稳定性。

这篇 paper 也和昨天那篇 AI-generated smells 很能接上。昨天看到的是，模型就算把东西写出来，也常常会留下维护层面的结构问题。今天看到的是，再把任务推到完整软件级别，很多模型连“先把整个房子盖起来”都还差得远。两个信号放在一起看，其实很清楚，AI coding 最大的短板已经越来越不像“写不出代码”，更像“不会长期地组织软件”。

这对做 code agent、repo agent、autonomous software building 的团队是个挺重要的提醒。接下来真正决定上限的，也许不是谁能多过几道小题，而是谁能把 agent 的架构决策、仓库组织能力、长期 planning 和 verification 机制做扎实。没有这些，演示视频里那种“从一句话长出一个产品”的感觉，很容易停在表面。

对 Nemo 这种关心 code capability、agent benchmark、软件工程真实约束的人来说，这篇值得优先读。它不是在增加想象力，它是在减少幻觉。

## 今日 3 篇精选

### 1) ProgramBench: Can Language Models Rebuild Programs From Scratch?
- 链接: https://arxiv.org/abs/2605.03546
- 摘要速读: 提出一个 200 任务 benchmark，让 agent 只看 executable 和文档，就要自己重建完整软件项目。结果 9 个模型无一完整解出任务，最好模型也只在极少数任务上逼近通过。
- 为什么重要: 它把 code agent 评价从局部修修补补，拉到了真正的软件构建层面。

### 2) OpenSeeker-v2: Pushing the Limits of Search Agents with Informative and High-Difficulty Trajectories
- 链接: https://arxiv.org/abs/2605.04036
- 摘要速读: 用更高难度、更有信息量的 trajectory 合成，加上更大的 knowledge graph 和 tool set，作者只靠 10.6k 条 SFT 数据就把 30B search agent 推到了新 SOTA。
- 为什么重要: 它给 search agent 训练提供了一个很有挑衅性的结论，很多时候更关键的是 trajectory 质量，不一定是更重的 CPT+RL 流水线。

### 3) ARISE: A Repository-level Graph Representation and Toolset for Agentic Fault Localization and Program Repair
- 链接: https://arxiv.org/abs/2605.03117
- 摘要速读: 给 repair agent 提供更细粒度的 repository graph 和 data-flow slicing 工具，在 SWE-bench Lite 上把定位指标和修复成功率都推高了一截。
- 为什么重要: 它说明 repo-scale code agent 的上限，很大程度取决于底层表示和可查询工具，而不只是上下文窗口有多大。

## 一句话结论
今天最强的研究信号是，**代码 agent 的难点已经明显落在软件结构这一层了。** benchmark 要更像真实项目，训练数据要更像高质量轨迹，工具层也要更懂 repository semantics。`2605.03546` 值得优先读，因为它把“从零做软件”这件事到底难在哪，说得最干脆。
