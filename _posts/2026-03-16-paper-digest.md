---
title: "Paper Digest: 2026-03-16"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天最值得盯住的信号，不是某个 agent 又把分数刷高了，而是训练 code agent 的基础设施开始变得像一个独立赛道。

## 今日 3 篇精选

### 1) daVinci-Env: Open SWE Environment Synthesis at Scale
- 链接: https://arxiv.org/abs/2603.13023
- 摘要速读: 这篇工作发布了 OpenSWE，一个面向 SWE agent 训练的开源环境构建框架。作者合成了 45,320 个可执行 Docker 环境，覆盖 12.8k 仓库，还把 Dockerfile、评测脚本、环境合成和过滤流程全部开源。更重要的是，它不是只堆规模，而是强调 dynamic feedback loop、难度过滤和可验证执行。
- 为什么重要: 现在 code agent 的瓶颈越来越不像“模型还不够聪明”，更像“训练和评测环境不够像真实世界”。这篇 paper 的价值，在于它把 benchmark 从静态数据集推进成了可制造、可执行、可筛选的基础设施。谁能持续生产高质量环境，谁就更可能在下一轮 agent 竞争里占优势。

### 2) Spend Less, Reason Better: Budget-Aware Value Tree Search for LLM Agents
- 链接: https://arxiv.org/abs/2603.12634
- 摘要速读: 作者提出 BAVT，一种 training-free 的 agent 推理搜索方法。核心思想很直接，agent 不该把 token 和 tool budget 当成无限资源，而要在推理过程中动态决定什么时候广撒网，什么时候收缩到最有希望的路径。
- 为什么重要: test-time scaling 这阵风吹得很猛，但真正落地时，预算、延迟和工具调用成本都是真约束。BAVT 的价值在于提醒大家，agent 的进步不一定来自更多 compute，也可能来自更聪明的 compute allocation。

### 3) ChainFuzzer: Greybox Fuzzing for Workflow-Level Multi-Tool Vulnerabilities in LLM Agents
- 链接: https://arxiv.org/abs/2603.12614
- 摘要速读: 这篇 paper 研究的是 multi-tool agent 的组合式漏洞。问题不在单个工具，而在一个工具产生的数据被另一个工具接着消费，最后形成可利用的数据流。作者在 20 个开源 agent app 上挖到了 365 个可复现漏洞，其中绝大多数需要多工具联动才能触发。
- 为什么重要: agent 越像 workflow system，安全问题就越不是 prompt injection 这么简单。真正需要防的是跨步骤、跨工具、带持久状态的攻击链。这个视角对任何想把 agent 接进生产流程的人都很关键。

## 可能感兴趣

### 4) LMEB: Long-horizon Memory Embedding Benchmark
- 链接: https://arxiv.org/abs/2603.12572
- 亮点: 给 memory-augmented system 做了一个更像真实长期记忆检索的 embedding benchmark，和传统 MTEB 呈现明显正交性。对做长期 memory agent 的团队有参考价值。

## 一句话结论
今天最强的信号来自 daVinci-Env。2026 年 code agent 的竞争，越来越像“谁能造出更好的训练环境”，而不是“谁能在同一套老 benchmark 上再多卷几个点”。
