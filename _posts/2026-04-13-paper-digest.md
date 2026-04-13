---
title: "Paper Digest: 2026-04-13"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的 paper，不在 Hugging Face 热榜里，而在 arXiv 的 cs.SE 角落。原因很简单，这篇 paper 碰到的是 code LLM 真正在生产里会撞上的墙，**模型知道的 API**，和**今天真实存在的 API**，经常已经不是同一个东西。**When LLMs Lag Behind: Knowledge Conflicts from Evolving APIs in Code Generation** 这篇 paper 把这个问题说得很清楚，RAG 确实能把新文档喂给模型，但当外部上下文和模型内部旧记忆冲突时，模型并不会自动乖乖听新文档的话。它会犹豫，会混用，会顽固地回到过时写法。

## 今日 3 篇精选

### 1) When LLMs Lag Behind: Knowledge Conflicts from Evolving APIs in Code Generation
- 链接: https://arxiv.org/abs/2604.09515
- 摘要速读: 这篇 paper 研究的是 software library 持续演化之后，code LLM 会怎么犯错。作者基于 8 个 Python 库的 270 个真实 API 更新，构建了一个 benchmark，覆盖 API deprecation、API modification 和 API addition 三类变化，然后测试了 4 个模型家族共 11 个模型。结果很扎心，没有完整文档时，生成代码在目标环境里的可执行率平均只有 42.55%。就算给了更结构化的文档、换更大的模型，可执行率也只有 66.36%。更有意思的是，reasoning-based strategy，比如 self-reflection，可以额外把 executable rate 往上拉 11 个点左右。这说明问题不只是 retrieval 不够，而是模型要处理一场更难的冲突，外部新知识和内部旧知识在打架。
- 为什么重要: 这篇 paper 最有价值的地方，是把一个很多人已经隐约感觉到的问题正式命名了，context-memory conflict。大家常说给模型接上 docs、接上 RAG、接上 MCP，就能解决 freshness 问题，但真实情况远没有这么顺。模型看到新文档，不代表它就能压住自己旧参数里那些已经习惯了的 API pattern。对 code agent 来说，这很关键，因为很多 bug 根本不是不会写，而是太“会写旧答案”。

### 2) SkillMOO: Multi-Objective Optimization of Agent Skills for Software Engineering
- 链接: https://arxiv.org/abs/2604.09297
- 摘要速读: 这篇 paper 讨论 coding agent 的 skill bundle 应该怎么自动优化。作者把 success rate、runtime、cost 一起当目标，用一个 optimizer agent 提出 skill 修改，再让 solver agent 跑任务评估，最后用 NSGA-II 做多目标筛选。结果显示，在三个 SkillsBench 任务上，pass rate 最高能提升 131%，成本还能下降 32%。
- 为什么重要: 它提醒人一件很实际的事，agent skill 不是越堆越好。很多系统最后都死在 prompt 膨胀上，说明多了，风格乱了，互相冲突了。SkillMOO 给出的信号很有意思，真正有效的优化常常来自 pruning 和 substitution，而不是继续累加说明书。

### 3) DeepGuard: Secure Code Generation via Multi-Layer Semantic Aggregation
- 链接: https://arxiv.org/abs/2604.09089
- 摘要速读: 这篇 paper 研究 secure code generation，核心判断是 vulnerability-related signal 并不只在 final layer，而是分布在中上层表示里。作者据此提出一个 multi-layer aggregation 框架，把多层语义聚合起来做 security analyzer，并在训练和推理阶段一起利用。实验报告说，相比强基线，secure-and-correct generation rate 平均提升 11.9%。
- 为什么重要: 它的意思很直接，安全问题未必只是数据标注或 final-layer steering 的问题，模型里可能早就有相关线索，只是最后一层为了 next-token prediction 把这些线索冲淡了。这个视角挺值得继续看。

## 一句话结论
今天最强的研究信号是，code intelligence 的真正难点越来越不是写出一段看起来像样的代码，而是让模型在一个持续变化的真实软件世界里，更新自己的判断、压住旧记忆、同时保持可靠性。API 在变，skills 会膨胀，安全信号藏在网络深处。真正有价值的系统，不只是会补 context，而是能在这些冲突里做出更稳的选择。
