---
title: "Paper Digest: 2026-04-14"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的 paper，不是又一个更大的 benchmark，而是一篇把 RL for LLMs 里一个很烦、但大家都见过的毛病讲清楚的工作。**The Past Is Not Past: Memory-Enhanced Dynamic Reward Shaping** 这篇 paper 盯住的是 rollout diversity 崩掉之后的那种熟悉场面，模型并不是不会探索，它只是会反复绕回同一种错误。表面上看每次采样都不完全一样，底层行为模式却高度重复。作者的判断很直接，光靠 entropy regularization 不够，因为它只是在当前策略下鼓励随机性，没有显式记住模型以前是怎么错过的。

## 今日 3 篇精选

### 1) The Past Is Not Past: Memory-Enhanced Dynamic Reward Shaping
- 链接: https://arxiv.org/abs/2604.11297
- 摘要速读: 这篇 paper 提出 MEDS，一种面向 RL for LLMs 的 memory-enhanced dynamic reward shaping 方法。核心做法是把历史 rollout 的中间表示存下来，用 clustering 找出那些高频重复出现的错误模式，再对落进这些错误簇的 rollout 施加更强惩罚。这样一来，模型不是被抽象地要求“更随机”，而是被具体地推离自己已经反复犯过的错。作者在 5 个数据集、3 个 base model 上报告，MEDS 相比已有方法平均都更强，最高能提升 4.13 个 pass@1 和 4.37 个 pass@128，同时 behavioral diversity 指标也更好。
- 为什么重要: 这篇 paper 最有启发性的地方，是它把 exploration 问题从“采样温度不够”改写成了“失败记忆不够”。很多后训练方法还是默认，只要给足随机性，模型自然会走出局部坑。但真实情况常常不是这样，模型会以不同表面措辞，重复同一类推理错误。MEDS 的价值就在于它承认这件事，并试着把“过去怎么错过”正式写进 reward shaping 里。

### 2) SkillMOO: Multi-Objective Optimization of Agent Skills for Software Engineering
- 链接: https://arxiv.org/abs/2604.09297
- 摘要速读: SkillMOO 把 coding agent 的 skill bundle 当作一个多目标优化问题来做，同时看 pass rate、runtime 和 cost。系统里有一个 optimizer agent 负责提出 skill 修改，一个 solver agent 负责跑任务评估，再用 NSGA-II 保留 Pareto 更优的组合。论文报告，在 3 个 SkillsBench software engineering 任务上，pass rate 最高提升 131%，成本最高下降 32%。
- 为什么重要: 它提醒人，agent system 的能力不是靠 prompt 越堆越厚。很多系统最后其实死在 skill 膨胀上，说明过多、重复、互相打架。SkillMOO 给出的结论很值得记，真正有效的优化往往来自 pruning 和 substitution，而不是继续往说明书上贴补丁。

### 3) DeepGuard: Secure Code Generation via Multi-Layer Semantic Aggregation
- 链接: https://arxiv.org/abs/2604.09089
- 摘要速读: DeepGuard 研究 secure code generation，先用 layer-wise probing 证明 vulnerability-related signal 在中高层最明显，到了 final layer 反而被冲淡，然后据此做 multi-layer aggregation，把多个上层表示聚合给一个 security analyzer，同时兼顾功能正确性。实验结果显示，相比强基线，secure-and-correct generation rate 平均提升 11.9%。
- 为什么重要: 它的意思很有意思，安全信号未必不存在，而是被 final-layer next-token objective 压扁了。这种看法对代码模型不只影响 security，也可能影响很多我们以为只能靠更多数据修的能力问题。

## 一句话结论
今天最强的研究信号是，LLM 行为改进越来越像一个“记忆与控制”问题。MEDS 让 reward shaping 记住模型以前掉过哪些坑，SkillMOO 让 agent skill 不再无限增生，DeepGuard 则去网络更深处找那些被输出层稀释掉的有用信号。真正有价值的系统，不只是会多采样，而是能更稳定地避开自己已经踩过的老坑。