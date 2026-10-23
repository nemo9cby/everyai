---
title: "Paper Digest: 2026-10-23"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Reinforcement Learning, Post-training, Distributed Systems]
---

今天最值得看的 paper 是 **IQuest-Q1: Advancing Foundation Capabilities for Agentic CLI Systems**。

IQuest-Q1 是一个面向 CLI agents 的 open-weight sparse MoE，参数总量 320B，每个 token 激活 15B。它的亮点集中在完整的 post-training stack：可执行任务如何合成，多个 agent harness 如何进入同一轮训练，环境故障如何与 policy failure 分开，以及多个 expert 的能力如何合并进一个 student。

论文报告了 63.0 的 NL2Repo、84.5 的 CyberGym、83.2 的 Terminal-Bench 2.1，以及 53.7 的内部 IQuest-CLIBench 成绩。更值得读的是这些结果背后的工程设计。

## 一条完整的 CLI-native 训练流水线

IQuest-Q1 的 pre-training 使用 Stable SplitNorm、Muon 和 muP，并在第二阶段提高 coding data 的比例。Mid-training 扩展数据领域、格式和 context length。Post-training 则包含两阶段 SFT、domain-specialized RL、multi-teacher on-policy distillation（MOPD）和 residual model merging。

Coding tasks 来自 GitHub repositories、PRs、issues、documentation、repository knowledge graphs，以及清洗后的使用模式。训练环境由多个版本的 Claude Code、Codex 和 OpenCode 驱动。Verifier 会组合 agent-written tests、经过调整的 repository tests 与 documented behavior checks。

每个 instance 还要通过 executability、leakage、instruction、reference solution、environment 和 verifier consistency 检查。Reference solution 必须通过，no-op solution 必须失败。作者继续用 trajectory review 过滤作弊和低质量执行，再用重复运行估计难度。

这部分揭示了 agentic RL 的核心成本：高质量 reward 依赖一套可以执行、可以隔离、可以追踪 provenance 的环境生产线。

## Multi-harness GRPO 如何控制训练方差

论文把 task domain、harness、harness version、environment 和 evaluator 都记录到 episode 中。共享 policy 会在不同 harness configurations 上收集 trajectories，但 GRPO 的相对比较保持同一个 task、固定 harness 和固定 evaluator version。

这个选择很重要。跨 harness 的 rollout 直接放进同一 group，会把 runtime 差异混入 advantage。IQuest-Q1 在全局采样层面平衡 domain-harness combinations，在 group 内部保持环境条件一致，并持续保留 regression cases。

Failure attribution 也发生在优化之前。Policy-caused errors 进入学习信号，和 policy 行为无关的 environment 或 harness faults 被 mask，evaluator fault 则阻止整条 sample 被消费。对于长 trajectory，segment-level rewards 只奖励可测量的 progress，同时惩罚无效计算、违反约束和 reward hacking。

## MOPD：让四个 expert 评价 student 自己的轨迹

MOPD 使用四个 frozen experts，分别覆盖 terminal user experience、multi-harness operation、long-horizon coding 与 cybersecurity、general agentic tasks。Student 先生成 on-policy rollouts，系统再按任务领域选择 teacher，让 teacher 对 student 已经生成的 token 计算 log probability，并形成 clipped token-level distillation advantage。

这种设计保留了 student 实际访问到的状态分布。Instructions 和 tool observations 只提供 context，loss 只作用于 model-generated tokens，其中包含 tool-call tokens。重复轨迹会被 mask，避免低质量循环污染蒸馏信号。

## 分布式系统细节决定 MOPD 能否扩展

作者实现了 fully asynchronous 的 distillation system，通过周期性同步 student parameters 和显式 sample-staleness control 限制 policy lag。Teacher forward pass 与 student log-probability computation 共用同一套 training engine path，减少执行差异。

多个 teachers 共享执行资源。Teacher weights 放在 distributed storage 中，只有当前使用的 shards 进入 GPU memory。下一个 teacher 的权重会在当前 forward pass 期间预取，并与计算重叠。这样增加 teacher 数量时，无需等比例增加常驻 GPU replicas。

这正是 post-training 与 heterogeneous distributed computing 的交叉点。训练算法定义了需要哪些 token-level signals，系统负责在有限 HBM、远端存储、异步 rollout 和不断变化的 student policy 之间维持吞吐与一致性。

## 对训练平台的四个直接启发

1. 每条 rollout 都要携带 task、harness、version、environment、evaluator 和 policy checkpoint 的完整 lineage。
2. GRPO grouping 要控制 runtime variation，跨 harness diversity 放在 sampling distribution 中处理。
3. Environment fault、evaluator fault 和 policy failure 需要独立归因，否则 reward 会训练模型适应基础设施噪声。
4. Multi-teacher distillation 的成本取决于 weight placement、prefetch、policy lag 和 execution-path consistency，teacher 数量只是表面变量。

## 今天另外两篇相关论文

**Stale Instruction Files Override the Code in Coding Agents: A One-Sentence Fix** 测试了 40 个 instruction file 与现有代码冲突的 Python projects。没有修复句时，agents 在 90.1% 的选择中服从 stale file。加入一条“现有代码优先”的规则后，code-consistent choices 从 4.2% 提升到 91.4%，test pass rate 从 89.4% 提升到 100%。

**Adaptive Skill Granularity for Coding Agents** 把 skill granularity 当成 runtime control policy。系统可以选择 primitive tool calls、macro skills 或 hybrid execution，并在 macro failure 后进入 micro-level recovery。它提醒 agent platforms 为 skill 定义 applicability、failure signals、intermediate state 和可验证的 recovery path。

论文：<https://arxiv.org/abs/2610.30949>

另外两篇：<https://arxiv.org/abs/2610.30571>、<https://arxiv.org/abs/2610.31461>
