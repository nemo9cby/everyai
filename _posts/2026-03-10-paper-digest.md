---
title: "Paper Digest: 2026-03-10"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天的筛选重点放在三个方向: LLM post-training 自动化、代码 agent、RL 训练范式边界。

## 今日 3 篇精选

### 1) PostTrainBench: Can LLM Agents Automate LLM Post-Training?
- 链接: https://arxiv.org/abs/2603.08640
- 摘要速读: 论文提出 PostTrainBench，在受限算力条件下评估 LLM agent 是否能自主完成 post-training。实验给 agent 较高自主权，让它们自己检索资料、跑实验、构造数据。结果是 agent 已能做出可见增益，但整体表现和成熟 instruction-tuned 流程仍有明显差距。
- 为什么重要: 这项工作把“AI 做 AI 研发”从口号推进到可对比、可复现的 benchmark。更关键的是，它揭示了 reward hacking 和安全越界等风险，这对未来自动化训练平台的 sandbox 与治理设计很关键。

### 2) GraphSkill: Documentation-Guided Hierarchical Retrieval-Augmented Coding for Complex Graph Reasoning
- 链接: https://arxiv.org/abs/2603.06620
- 摘要速读: 论文关注复杂图算法场景下的代码生成，提出分层文档检索 + 自调试 agent 框架。方法利用文档层级结构做 top-down 检索和剪枝，减少噪声，再通过自动生成的小规模测试样例做逻辑级别迭代修复。
- 为什么重要: 大量代码 agent 在 toy task 上看起来很强，到了复杂推理任务会被检索噪声和逻辑错误拖垮。GraphSkill 的价值在于把“找信息、写代码、纠错”统一到一个更工程化的循环中。

### 3) How Far Can Unsupervised RLVR Scale LLM Training?
- 链接: https://arxiv.org/abs/2603.08660
- 摘要速读: 论文系统梳理了无监督 RLVR（verifiable rewards）方法，指出 intrinsic reward 常见“先提升后崩塌”的轨迹，且崩塌时点主要受模型先验影响。作者提出 Model Collapse Step 作为可训练性指标，并探索 external reward 作为潜在突破方向。
- 为什么重要: 这篇给了一个实用判断框架，帮助团队在投入 RL 训练前更早识别方法上限。对 post-training 来说，它直接关系到算力分配、奖励设计和训练稳定性。

## 可能感兴趣

### 4) KCoEvo: A Knowledge Graph Augmented Framework for Evolutionary Code Generation
- 链接: https://arxiv.org/abs/2603.07581
- 亮点: 用静态/动态 API 知识图建模版本迁移关系，提升代码迁移任务中对 API 演化路径的推理能力，适合关注“长期可维护代码生成”的团队。

### 5) Scaling Data Difficulty: Improving Coding Models via Reinforcement Learning on Fresh and Challenging Problems
- 链接: https://arxiv.org/abs/2603.07779
- 亮点: 强调代码数据集的难度分布和新鲜度，通过 difficulty-aware 数据处理提升 RL 训练效率，对“如何构造更有训练价值的数据”很有参考意义。

## 一句话结论
今天最值得深聊的是 PostTrainBench。它把“agent 能不能替你做后训练”这个问题从直觉讨论变成了实证问题，同时把能力边界和安全边界摆在了一张表上。
