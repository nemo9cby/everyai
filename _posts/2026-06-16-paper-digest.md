---
title: "Paper Digest: 2026-06-16"
categories: [Paper Digest]
tags: [AI, Coding Agents, SWE Bench, Post Training, Agent Harness]
---

今天最值得看的 paper，我会选 **Open-SWE-Traces**: `Open-SWE-Traces: Advancing Dual-Mode Multilingual Distillation for Software Engineering Agents`。

这篇论文抓住了 open coding agent 最现实的瓶颈：不是缺一个更漂亮的 demo，而是缺足够大的、足够真实的、license 能用的长轨迹数据。

论文发布了一个 dataset：207,489 条 agentic software engineering trajectories，来自 20,000 个真实 pull requests，覆盖 Python、Go、TypeScript、JavaScript、Rust、Java、PHP、C、C++ 九种语言。轨迹通过 OpenHands 和 SWE-agent harness 收集，并且做了 permissive license 过滤，来源是 SWE-rebench-V2 里的 MIT、Apache、BSD repo。

比较有意思的是，它不是只收一种 trace。

作者做了 dual-mode synthesis：一类是 Minimax-M2.5 生成的 explicit thinking trajectories，另一类是 Qwen3.5-122B 生成的 non-thinking traces。也就是说，它同时考虑了"带思考过程的 agent 轨迹"和"不暴露 thinking 的执行轨迹"。

这对 post-training 很关键。

很多 coding-agent 训练讨论会卡在一个问题上：到底应该教模型完整 thought process，还是只教 action / observation / patch / test 这种外部可见过程？Open-SWE-Traces 没有把这个问题抽象成理念争论，而是直接给出两种数据模式，让模型训练和 benchmark 结果来回答一部分。

实验部分也够硬。作者 fine-tune 了 Qwen3-30B-A3B 系列，包括 Thinking、Instruct、Coder 变体。最好的模型在 SWE-bench Verified 上达到 61.7%，SWE-bench Multilingual 上 57.1%，SWE-bench Pro 上 36.8%。

这些数字最值得注意的地方，不只是分数本身，而是它说明开源模型的 SWE-agent 能力可以通过大规模 trace distillation 往前推。

今天另外两篇可以和它放在同一条线上看。

**LLM-as-Code Agentic Programming for Agent Harness** 讨论的是 agent harness 架构。它的观点很直接：循环、分支、终止、sequencing 这些确定性的控制流，不应该都交给 LLM。程序负责 control flow，LLM 只在需要 reasoning 或 generation 的地方被调用。

这个设计会让 context 从 execution history 的 call tree 里构建出来，形成 DAG。于是每次调用的上下文长度由 call depth 决定，而不是随着 step 一路线性膨胀。对 long-horizon computer-use agent 来说，这个思路很实用。

**Agent trajectories as programs** 则看 agent 轨迹本身。论文发现，coding agents 有可识别的 procedural fingerprints。一个 probe 可以用 unseen trajectories 以 85.7% 的准确率识别出是哪一个 agent，而且控制了 task leakage。

这很重要。SWE-bench 分数告诉你一个 agent 过了多少题，但很难告诉你它怎么过的。它是先定位测试？先 grep？先改核心逻辑？遇到失败会回滚还是硬修？这些过程差异，可能决定了成本、稳定性和可迁移性。

把三篇放在一起，今天的信号很清楚：

coding agent 的下一步不只是模型更强。它需要更好的 trace data，需要更清晰的 harness control flow，也需要能分析 procedure 的 eval 方法。

如果你在做 coding-agent post-training，Open-SWE-Traces 值得优先读。它不是一个纯 benchmark paper，而是一条可以复用的训练数据路线：真实 PR、agent harness、双模式 trace、license 过滤、多语言覆盖、SWE-bench 验证。

## 今日 3 篇精选

### 1) Open-SWE-Traces: Advancing Dual-Mode Multilingual Distillation for Software Engineering Agents
- 链接: https://arxiv.org/abs/2606.16038
- 摘要速读: 发布 207,489 条 agentic SWE trajectories，来自 20,000 个真实 PR，覆盖九种编程语言，并用 thinking / non-thinking 双模式做 distillation。
- 为什么重要: open coding agent 的 post-training 需要大规模、真实、可复用的长轨迹数据，这篇给了一条很具体的路线。

### 2) LLM-as-Code Agentic Programming for Agent Harness
- 链接: https://arxiv.org/abs/2606.15874
- 摘要速读: 提出 Agentic Programming，让程序掌控 loops、branches、sequencing，LLM 只在需要 reasoning 或 generation 的节点被调用。
- 为什么重要: 很多 agent 不稳定来自把确定性控制流交给 probabilistic model；harness 设计本身就是能力边界。

### 3) Agent trajectories as programs: fingerprinting and programming coding-agent behavior
- 链接: https://arxiv.org/abs/2606.16988
- 摘要速读: 把 coding-agent trajectories 表示成 procedures，发现不同 agent 有可识别的行为指纹，并提出 ProcGrep 做过程审计。
- 为什么重要: benchmark pass rate 只看结果，procedural fingerprint 能看 agent 的搜索、编辑、验证和恢复风格。

## 一句话结论

今天最强的研究信号是：**coding agent 的核心资产正在变成 trace data、harness control flow 和 procedure-level evaluation。** `2606.16038` 值得先读。
