---
title: "Paper Digest: 2026-03-13"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天这批 paper 里，我更在意一个很现实的问题: agent 的能力到底有没有开始碰到真正值钱的工程工作，而不只是把 demo 做得更顺。

## 今日 3 篇精选

### 1) Resolving Java Code Repository Issues with iSWE Agent
- 链接: https://arxiv.org/abs/2603.11356
- 摘要速读: iSWE Agent 专门做 Java 仓库 issue resolution，这一点本身就很关键。大多数 SWE agent 的亮眼结果都建立在 Python 友好生态上，但企业里大量关键系统依然是 Java。作者把流程拆成 localization 和 editing 两个子 agent，并给它们配上 Java 静态分析和代码变换工具，在 Java 版 Multi-SWE-bench 和 SWE-PolyBench 上拿到新 SOTA。
- 为什么重要: 这篇最值得看的地方，不只是 benchmark 涨了，而是它开始对准真实企业软件环境。代码 agent 下一阶段如果要落地，拼的不会只是模型会不会写代码，而是能不能进入强类型、重规则、可维护性优先的代码库。

### 2) Strategic Navigation or Stochastic Search? How Agents and Humans Reason Over Document Collections
- 链接: https://arxiv.org/abs/2603.12180
- 摘要速读: 作者提出 MADQA，用 800 份异构 PDF 和 2250 道人工问题去测 agent 在复杂文档集合上的搜索与推理能力。结果很有意思，最强 agent 的原始准确率已经能接近人类，但答对的题目分布和人类差异很大，而且大量依赖 brute-force search，反复陷入低效循环，和 oracle 还有接近 20% 的差距。
- 为什么重要: 这篇 paper 的价值在于它替很多热闹叙事泼了一盆冷水。很多所谓“agent 已经能做研究、读文档、做分析”的展示，背后可能并不是更成熟的 reasoning，而是更贵、更慢的搜索。这个区别，决定了 agent 到底是能扩展的生产力工具，还是一台昂贵的试错机器。

### 3) Automatic Generation of High-Performance RL Environments
- 链接: https://arxiv.org/abs/2603.12145
- 摘要速读: 这篇工作展示 coding agent 可以把 RL environment 翻译成高性能实现，甚至从规范中合成新环境。方法核心是 prompt template、分层验证和迭代修复，重点不只是“写出来”，而是还要做 property test、interaction test、rollout test，确认语义等价。作者给出的速度提升很夸张，尤其在 Pokemon battle simulator 这种复杂环境上。
- 为什么重要: 现在很多 agent demo 还停留在 landing page、脚手架、简单应用层。如果 agent 开始稳定进入模拟器、训练环境、基础设施代码这一层，影响就完全不同了。因为这里压缩的是研究和训练系统里的高价值工程劳动。

## 可能感兴趣

### 4) QUARE: Multi-Agent Negotiation for Balancing Quality Attributes in Requirements Engineering
- 链接: https://arxiv.org/abs/2603.11890
- 亮点: 把 requirements engineering 变成多 agent 协商问题，让不同质量属性显式谈判。更像“agent 化软件工程流程”的一个方法论信号。

## 一句话结论
今天最强的信号来自 iSWE Agent 和 MADQA。前者说明 code agent 正在往企业真实栈里钻，后者说明很多 agent 离“真的会推理”还差得远。把这两件事放在一起看，会更接近 2026 年 agent 的真实状态。 
