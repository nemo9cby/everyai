---
title: "Paper Digest: 2026-03-27"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天最值得看的 paper，不是某个模型又把 benchmark 顶高了多少，而是有人终于认真量化了一个大家早就隐约知道、却一直没有被主流评测正面处理的问题：coding agent 在长程迭代里，会把代码越写越“sloppy”。第一次交卷能过，不代表第五轮需求改完之后，这个系统还像个能维护的工程。

## 今日 3 篇精选

### 1) SlopCodeBench: Benchmarking How Coding Agents Degrade Over Long-Horizon Iterative Tasks
- 链接: https://arxiv.org/abs/2603.24755
- 摘要速读: 这篇 paper 把 coding agent evaluation 从 single-shot 拉回真实软件开发。作者设计了 20 个问题、93 个 checkpoint，让 agent 在不断变化的规格下，持续接着自己之前写出的代码往前改。结果很扎心，没有任何模型能完整走完全程，最高 checkpoint solve rate 只有 17.2%。更关键的是，代码质量会持续恶化，verbosity 在 89.8% 的轨迹里上升，structural erosion 在 80% 的轨迹里上升。和 48 个开源 Python 仓库相比，agent 代码平均冗长 2.2 倍。
- 为什么重要: 这篇 paper 真正击中的，不是“agent 代码写得丑”这种表面抱怨，而是 benchmark 本身在奖励错误的东西。只看单次 pass rate，会天然忽略架构退化、重复逻辑堆积、复杂度向坏函数集中这些会在后续迭代里炸掉的结构性问题。对 code agent 来说，真正困难的从来不是第一版，而是第 N 版还活得像个工程。

### 2) Revisiting On-Policy Distillation: Empirical Failure Modes and Simple Fixes
- 链接: https://arxiv.org/abs/2603.25562
- 摘要速读: 这篇 paper 重新审视 on-policy distillation 在长轨迹 post-training 里的脆弱性。作者认为常见的 sampled-token 方案把 sequence-level 监督过度简化成单 token 信号，rollout 一旦偏离 teacher 熟悉的 prefix，teacher guidance 就会迅速失真。paper 不只指出问题，还从 estimator 和实现层面分析偏差与方差的 tradeoff，并用 top-K local support matching 给出更稳定的替代实现。
- 为什么重要: 对做 post-training 的人，这篇比“某模型又涨了几分”更值钱。它讨论的是训练为什么会不稳，为什么 long-horizon setting 下看起来合理的监督信号会突然失效。很多 agent 和 reasoning 系统最后卡住，往往不是因为想法不够新，而是训练链路里某个看似无害的近似在长轨迹条件下失真得太厉害。

### 3) FinMCP-Bench: Benchmarking LLM Agents for Real-World Financial Tool Use under the Model Context Protocol
- 链接: https://arxiv.org/abs/2603.24943
- 摘要速读: FinMCP-Bench 给 MCP agent 提供了一个更像真实工具使用环境的 benchmark，包含 613 个样本、65 个真实 financial MCP，覆盖 single-tool、multi-tool、multi-turn 三类任务。虽然场景在金融，但它真正有意思的地方，是把 agent 对 MCP 工具生态的使用能力变成了更系统的测量对象。
- 为什么重要: MCP 现在已经是 agent infra 世界里最重要的接口层之一，但 benchmark 还远没跟上。FinMCP-Bench 的价值，在于它让“会不会调工具”这件事离开 demo 层，开始进入可比、可测、可复现的评估框架。

## 一句话结论
今天最强的研究信号是，coding agent 的问题已经不只是“能不能写出能跑的代码”，而是“会不会在连续修改里把系统一步步写烂”。如果 benchmark 还只看单次正确率，很多真正决定生产可用性的失败，根本不会被统计进去。
