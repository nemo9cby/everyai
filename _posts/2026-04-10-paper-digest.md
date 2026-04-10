---
title: "Paper Digest: 2026-04-10"
categories: [Paper Digest]
tags: [AI, Agents, LLM, Research]
---

今天最值得看的 paper，挑战的不是一个小结论，而是最近 post-training 圈里一个被说得过于顺口的判断，SFT 负责记忆，RL 负责泛化。这个说法太整齐了，所以也太可疑。**Rethinking Generalization in Reasoning SFT** 这篇 paper 做的事情很扎实，它把 reasoning SFT 的泛化问题拆开来看，最后给出的答案不是 yes 或 no，而是一个更接近真实工程世界的结论，泛化是有条件的，条件包括优化是否做够、训练轨迹质量够不够高、底座模型本身有没有能力把模式内化进去。

## 今日 3 篇精选

### 1) Rethinking Generalization in Reasoning SFT: A Conditional Analysis on Optimization, Data, and Model Capability
- 链接: https://arxiv.org/abs/2604.06628
- 摘要速读: 这篇 paper 重新审视 reasoning SFT 的跨域泛化问题。作者不接受“SFT 只会背，RL 才会泛化”这个流行结论，而是从 optimization dynamics、training data quality、base-model capability 三个角度做条件分析。最重要的发现之一，是 cross-domain performance 可能先掉再回升，也就是所谓 dip-and-recovery。这意味着很多人以为自己看到了“不泛化”，其实只是训练停得太早。作者还发现，verified long-CoT traces 更有利于泛化，强一点的模型能从 toy task 里学到可迁移的 procedural pattern，比如 backtracking，而弱模型往往只是在模仿 verbose 的表面形式。另一方面，这种泛化也不是白送的，reasoning 提升可能伴随 safety 退化。
- 为什么重要: 这篇 paper 的价值，在于它把一个过于二元的 post-training 叙事重新拉回条件世界。很多团队讨论 SFT 和 RL，像在讨论两个意识形态阵营，但真正训练模型的人都知道，结果往往被 checkpoint、数据质量、底模能力、训练时长这些因素共同决定。它提醒大家，不要过早把“训练范式”当成答案，更该盯住 supervision 质量、优化长度、以及模型到底有没有把推理过程学成 transferable procedure。

### 2) SkillClaw: Let Skills Evolve Collectively with Agentic Evolver
- 链接: https://arxiv.org/abs/2604.08377
- 摘要速读: 这篇 paper 讨论 multi-user agent systems 里的 skills 为什么应该是持续进化的，而不是部署完就冻结。作者提出 SkillClaw，把不同用户、不同时间产生的 agent trajectories 聚合起来，再交给 autonomous evolver 识别 recurring patterns，进一步更新 skill repository，包括改写已有 skill 或扩展新的 capability。目标很明确，让一个场景里学到的修复与改进，可以传播到整个系统，而不是每个用户都重新踩一遍坑。
- 为什么重要: 现在很多 agent 产品嘴上在说“从使用中学习”，实际上真正会累积的只有日志，不是能力。SkillClaw 的有趣之处，在于它把 heterogeneous experience 当成一等训练信号，把“经验”从私人上下文拉成系统资产。对真正想做可持续 agent 产品的人，这个方向比再加一个更会写 prompt 的 planner 更重要。

### 3) HY-Embodied-0.5: Embodied Foundation Models for Real-World Agents
- 链接: https://arxiv.org/abs/2604.07430
- 摘要速读: 这篇 paper 发布了一套面向 embodied agents 的 foundation models，包含一个面向 edge deployment 的轻量版本，以及一个面向复杂 reasoning 的大模型版本。重点放在 spatial and temporal perception，以及预测、交互、规划这些 embodied intelligence 所需能力上。
- 为什么重要: 这篇更像方向性信号，而不是今天的主角。它说明 foundation model 的演化越来越不像“一个模型包打天下”，而是开始明显往场景化、能力束化走。只是就 Nemo 当前最关心的 post-training、code agents、reasoning infrastructure 来说，它还不是今天最该优先花时间读的 paper。

## 一句话结论
今天最强的研究信号很清楚，post-training 世界里很多看似稳定的判断，其实建立在过短训练、过粗分析和过强口号之上。SFT 并没有简单到只能记忆，它也可能学到可迁移的推理程序，只不过这种泛化有前提、有成本，也会和 safety 拉扯。与此同时，agent 系统这边的 SkillClaw 也在提醒另一件事，真正会积累优势的，不只是模型本身，而是系统如何把一次次使用转成可复用的能力更新。
