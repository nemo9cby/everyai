---
title: "Paper Digest: 2026-09-29"
categories: [Paper Digest]
tags: [AI, LLM, Code Agents, Reinforcement Learning, GRPO, Post-training]
---

今天最值得看的 paper 是 **Groupwise Agentic Grading and Advantage Redistribution for Code Agent RL**，简称 GAGAR。

代码智能体做 RL 时，最方便的 reward 来自测试：通过为 1，失败为 0。这个信号客观、便宜、容易扩展，却把所有成功轨迹压成了同一种结果。

同一个 issue，可以有两份都能通过测试的 patch。一份找准 root cause，只改必要的几行；另一份绕开约束、扩大修改范围、加入多余逻辑，甚至留下 hidden side effects。普通 GRPO 在组内只看到相同的 binary reward，因此给它们完全相同的 positive advantage。

GAGAR 想解决的问题很具体：**当多个 coding trajectories 都通过测试时，怎样让更精准、更克制、更接近 merge-ready 的实现得到更多训练信用？**

## 让 grader 进入真实 workspace

GAGAR 为同一个任务采样多条轨迹，只保留同时包含成功和失败样本的 mixed-outcome groups。每一组被放进共享 workspace，交给一个 SFT-trained agentic grader。

Grader 可以查看：

- task specification
- 完整 agent trajectories
- 每条轨迹提交的 patch
- repository source code
- test output 和 execution evidence

它还能读取相关文件、交叉检查修改，并在证据不足时运行 targeted checks。负面评价必须引用 patch location、trajectory event 或 execution result，避免仅凭代码风格做主观判断。

成功实现会从五个维度进行组内排名：

1. solution approach 是否合适
2. implementation 是否精准
3. changes 是否足够 minimal
4. 是否避免 unintended side effects
5. 是否遵循 codebase conventions

组内比较很重要。面对同一个 task 和同一份初始 repository，grader 更容易判断哪些改动确实必要，也能用失败轨迹识别无效的排查路径。

## 为什么要保存 positive advantage 总量

拿到质量排名后，最直接的做法是降低较差成功轨迹的 advantage。论文发现，这会破坏原本的 credit balance。

GAGAR 先按排名降低低质量 pass 的权重，再把所有 passing trajectories 的 advantage 按比例缩放，使 positive advantage 的总和回到原值。失败轨迹的 advantage 保持不动。

结果是组内信用发生重新分配：高质量实现获得更多，低质量实现获得更少，整组成功样本相对于失败样本的训练强度不变。

这个 conservation constraint 看起来像一个优化细节，实际决定了训练能否稳定。Downweighting-only ablation 中，policy entropy 从 0.359 升到 0.905，平均 rollout length 从 47.1K tokens 增长到 114.1K。使用 sum-preserving redistribution 后，entropy 只升到 0.513，平均长度为 69.9K tokens。

在 DeepSWE 上，downweighting-only 的 pass rate 一度从 56.5% 跌到 48.8%；完整方法在相同后期 checkpoint 达到 62.2%。丰富 reward signal 时，信用的总量和相对平衡同样需要被控制。

## 工业规模实验

论文从两组 pre-RL SFT checkpoints 开始训练：

- MiMo-V2.6-Flash：310B total parameters，15B active
- MiMo-V2.6-Pro：1.02T total parameters，42B active

主要 controlled experiment 使用 Flash、每个 prompt 16 条 rollouts，并在 DeepSWE v1.1 和 SWE-bench Pro 上比较 ordinary binary-reward training 与 GAGAR。

在共享的 DeepSWE checkpoint，GAGAR 把平均交互轮数从 132.3 降到 111.6，减少 15.6%；平均 token length 从 191.9K 降到 172.9K，减少 9.9%。在 SWE-bench Pro 上，平均轮数从 58.4 降到 54.6，token length 从 79.9K 降到 68.6K。

更短的轨迹并未牺牲 patch quality。对通过测试的实现做 blinded evaluation 时，GAGAR 的平均质量分为 4.03，binary-reward baseline 为 3.70；它在 pairwise comparison 中取得 69.8% average win rate，并在 65.0% 的 rollout groups 中排名第一。

Mixed-task RL 后，Flash 和 Pro 在 DeepSWE v1.1 上的 avg@3 分别达到 67.9 和 71.9，说明这套方法可以进入包含多领域任务的大规模训练流程。

## 对 code-agent RL 的启发

GAGAR 给出了一套可以复用的训练设计：

- executable tests 继续负责可靠的 outcome verification
- agentic grader 负责比较成功实现之间的质量差异
- comparison 在同一 task 的 rollout group 内完成
- grader 接触 repository 和 execution evidence，而非只看最终回答
- positive credit 在成功轨迹之间重新分配，并保持总量

对实际 post-training pipeline，评估指标也应该超出 pass rate。值得同时监控 turns、tokens、truncation、entropy、patch scope、unintended side effects，以及成功 patch 需要多少人工 review 才能合并。

这篇 paper 与 Nemo 的工作交集很直接：code agents、GRPO、post-training 和大规模 rollout systems。一个很自然的内部 ablation，是在相同 rollout data 上比较 ordinary GRPO、quality downweighting 和 sum-preserving redistribution，再核算异步 grading 的成本能否由更短轨迹、更稳定训练和更高 patch quality 抵消。

## 今天另外两篇相关论文

**Post-Training Leaves Behavioral Shadows on Unrelated Decisions** 提出 Active Taskless Distillation。它只向 post-trained teacher 查询普通、无关 prompt 上的单词选择，用 5,664 个 one-word responses 让 Qwen2.5-1.5B student 在 HumanEval+ 上超过 matched control 5.34 个百分点。这意味着 off-task black-box behavior 也可能泄露 capability-relevant information。

**WideSWE** 收集 120 个需要跨 repository 协同修改的真实任务。七种 agent configurations 的完整任务成功率只有 10.83% 到 42.50%。Codex CLI 与 GPT-5.6-sol 表现最好，但 69 个失败任务中有 48 个出现“部分 repositories 已通过、其余仍失败”，揭示 scope discovery 和跨 repo 完整性检查仍是明显短板。

论文：<https://arxiv.org/abs/2609.32577>

另外两篇：<https://arxiv.org/abs/2609.29233>、<https://arxiv.org/abs/2609.33382>
