---
title: "Paper Digest: 2026-04-08"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的 paper，不在于它又给 agent 加了什么新能力，而在于它终于认真去量 agent 到底做对了什么、做错了什么。过去很多 autonomous agent benchmark 最大的问题，是只看终点，不看过程。最后按钮点对了、文件改对了、答案交对了，分就给了。但真实世界里的 agent，往往恰恰死在过程里，绕远路、碰安全红线、靠运气撞对结果。Claw-Eval 这篇 paper 把这层遮羞布扯开了。

## 今日 3 篇精选

### 1) Claw-Eval: Toward Trustworthy Evaluation of Autonomous Agents
- 链接: https://arxiv.org/abs/2604.06132
- 摘要速读: 这篇 paper 提出一个面向 autonomous agents 的 end-to-end evaluation suite，覆盖 300 个 human-verified tasks，横跨 service orchestration、multimodal perception and generation、professional dialogue 三大类。核心改动很重要，它不再只看 final output，而是把 agent 的执行轨迹完整记下来，包括 execution traces、audit logs、environment snapshots，再从 completion、safety、robustness 三个维度做细粒度打分。作者一共设计了 2,159 个 rubric items，并且做多次 trial，用 Pass@k 和 Pass^k 区分“真的稳定能做”还是“偶尔撞上了”。
- 为什么重要: 这篇 paper 的价值非常直接，它把 agent eval 从 demo-friendly 拉回 deployable。很多 benchmark 看起来分数很高，只是因为它们默认 final-state correctness 足够代表能力。但 agent 一旦进入真实软件环境，中间每一步都可能出问题。Claw-Eval 给出的结果也很扎眼，trajectory-opaque evaluation 会漏掉 44% 的 safety violations 和 13% 的 robustness failures。这不是小修小补，这是在提醒整个 agent 研究社区，很多“看起来能用”的系统，其实只是被 eval 宽容地放过去了。

### 2) Learning to Retrieve from Agent Trajectories
- 链接: https://arxiv.org/abs/2604.04949
- 摘要速读: 这篇 paper 讨论 retrieval 在 agent 时代应该怎么重新训练。传统 IR 模型主要用 human click logs 学 relevance，但 agent 发 query、读文档、放弃候选、整合证据的方式，和人类完全不是一回事。作者提出 LRAT，从 agent 的 multi-step trajectories 里挖 supervision，包括浏览行为、未浏览但拒绝的文档、以及 post-browse reasoning traces，用这些信号训练 retrieval model。实验显示，这种 retriever 能提升 evidence recall、end-to-end task success 和 execution efficiency。
- 为什么重要: 很多 deep research agent 的失败，表面看是 reasoning 问题，底层其实是 retrieval layer 根本没有按 agent 的消费方式来训练。这篇 paper 提醒得很准，agent 不是人类用户的替代品，它会产生另一种 query distribution，也会暴露另一种 utility signal。如果 retrieval 还停留在人类点击逻辑上，整个 agent stack 会从根上带偏。

### 3) ACES: Who Tests the Tests? Leave-One-Out AUC Consistency for Code Generation
- 链接: https://arxiv.org/abs/2604.03922
- 摘要速读: 这篇 paper 盯住 code generation 里一个很实际的问题，很多系统会用 LLM 生成 tests，再拿这些 tests 去 rank 多个 code candidates。问题是 tests 自己也可能是错的。ACES 的思路很漂亮，它不强行判断哪个 test 是“真的对”，而是问一个更有用的问题，这个 test 能不能把更好的代码和更差的代码区分开。作者用 leave-one-out AUC consistency 给 tests 加权，在多个 benchmark 上拿到了更好的 Pass@k。
- 为什么重要: 这是一类很容易被忽视、但实际上离 production 很近的问题。合成 tests 从来不是完全可信的，可很多 pipeline 默认它们投票等权。ACES 给出的不是花哨的框架，而是一个更稳的 ranking logic，尤其适合 code candidate reranking 和 synthetic test-heavy 的系统。

## 一句话结论
今天最强的研究信号很清楚，agent 系统真正缺的，不只是更强的 final answer，而是更可信的过程建模。Claw-Eval 让轨迹本身进入评价核心，LRAT 让 retrieval 学会从 agent 的真实行为里吸收监督，ACES 则在 code generation 里重新定义什么叫“有用的测试”。如果说过去大家在追逐会做事的 agent，接下来更值得追的是，知道 agent 是怎么做事、哪里做错、为什么偶尔做对。