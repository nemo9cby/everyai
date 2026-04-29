---
title: "Paper Digest: 2026-04-29"
categories: [Paper Digest]
tags: [AI, Coding Agents, Observability, Evaluation, Post-Training]
---

今天最值得看的 paper，我会给 **Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses**。原因很直接，它抓住了 coding agent 这波里最容易被低估、但又最有杠杆的那一层，**真正决定 agent 上限的，常常不只是 model，本质上还是 harness。**

这篇 paper 讨论的不是“再换一个 prompt 会不会更好”，也不是“再上一个更强模型是不是就行”，而是把 attention 放到 agent 和 repo、tools、execution environment 之间那层工程界面上。作者把这件事叫做 Agentic Harness Engineering，简称 AHE。核心观点很清楚，harness 不是静态脚手架，而是一个可以被持续观测、持续修改、持续验证的系统对象。

AHE 做了三层 observability。第一层是 **component observability**，把每个可编辑 harness 组件都明确成 file-level representation，意味着 action space 是显式的、可回退的。第二层是 **experience observability**，把动不动就几百万 token 的 agent trajectory 压缩成分层证据语料，让后续迭代真的能消费这些经验，而不是被日志淹没。第三层是 **decision observability**，要求每一次 harness edit 都配上自我预测，再拿下一轮任务结果去验证，相当于把每次修改都变成一个可证伪的 contract。

我觉得它最有意思的地方，在于它把“agent 调不好”这件事从玄学往工程学拉了一大步。很多 coding agent 系统目前都有类似问题，大家知道 harness 很重要，但改 harness 的方式仍然很像黑箱试错。一个参数换掉，一个工具顺序挪一下，一个上下文裁剪策略调一调，最后结果可能变好，也可能只是 benchmark lucky hit。AHE 想做的是把这个循环结构化，让 harness evolution 本身拥有证据、归因和反馈闭环。

结果也足够硬。paper 里说，经过 10 次 AHE 迭代，Terminal-Bench 2 的 pass@1 从 **69.7% 提升到 77.0%**，超过 human-designed 的 Codex-CLI harness（71.9%）。更重要的是，它不是只在一个 benchmark 上抠分。冻结后的 harness 迁移到 SWE-bench-verified 还能保持更好的 aggregate success，并且 token 更省。在 Terminal-Bench 2 上，它对三种其他模型家族还能带来 **+5.1 到 +10.1 个点**的跨家族提升。这说明进化出来的并不只是 benchmark hack，更像是可迁移的工程经验。

这篇 paper 对今天的 agent 研究有一个很现实的提醒。很多人还把 agent 进步理解成模型越来越聪明，但生产环境里更常见的真相是，**系统设计、工具界面、可观测性和反馈回路，往往比额外那一点 base model gain 更值钱。** 尤其是 coding agent，这本来就是一个被 harness 深度塑形的任务。你给它什么文件视图，怎么暴露工具，怎样切摘要层次，如何验证每一步 edit，最后都会直接反映成成功率、token 成本和稳定性。

当然，这种方向也有一个天然问题，复杂度会不会越来越高，最后把 harness 自己变成一个难以维护的新系统。我觉得这是值得继续观察的点。但至少这篇 paper 没停在概念层，它给了清楚的方法框架，也给了有说服力的结果。所以如果你最近在看 coding agents，这篇值得优先读。

## 今日 3 篇精选

### 1) Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses
- 链接: https://arxiv.org/abs/2604.25850
- 摘要速读: 提出 AHE，用 component observability、experience observability 和 decision observability 三层机制，自动推进 coding-agent harness 的持续进化。
- 为什么重要: 它把 coding agent 的关键问题从“模型够不够强”重新拉回“系统外壳怎么设计、怎么迭代、怎么验证”。这是更接近真实工程杠杆的位置。

### 2) Recursive Multi-Agent Systems
- 链接: https://arxiv.org/abs/2604.25917
- 摘要速读: 把 multi-agent collaboration 建模成 latent-space recursive computation，用 RecursiveLink 连接异构 agent，并做 whole-system co-optimization。
- 为什么重要: 它代表一种更激进的方向，agent 协作不再只是文本通信，而是被当成可以整体学习和整体加深的计算过程。

### 3) Programming with Data: Test-Driven Data Engineering for Self-Improving LLMs from Raw Corpora
- 链接: https://arxiv.org/abs/2604.24819
- 摘要速读: 把训练数据当 source code，把 benchmark 当 unit test，把 failure-driven data repair 当 debugging，尝试把 post-training 变成可追踪、可修补的数据工程闭环。
- 为什么重要: 这篇 paper 很适合做 post-training 的人看，因为它提供了一个更像工程学的语言去描述 dataset iteration，而不是继续盲目堆数据。

## 一句话结论
今天最强的研究信号是，**agent 的真正进步越来越依赖模型外部的工程层。** `2604.25850` 值得重点看，因为它把 harness 从配角推到了主角位置。