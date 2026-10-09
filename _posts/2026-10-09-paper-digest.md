---
title: "Paper Digest: 2026-10-09"
categories: [Paper Digest]
tags: [AI, LLM, Reinforcement Learning, Coding Agents, Post-training, Distributed Systems]
---

今天最值得看的 paper 是 **MiMo-V2.6: Scaling Reinforcement Learning Towards Self-Improvement**。

这份小米技术报告给出的关键信号很直接：训练更强的 agent model，需要同时扩展 rollout、grader、environment、harness 和 training infrastructure。只扩大优化器一侧的计算，撑不起长轨迹、多任务、大 batch 的 agentic RL。

MiMo-V2.6 包含两个 sparse-MoE 模型：Pro 版总参数 1.02T、激活参数 42B，Flash 版总参数 310B、激活参数 15B。团队在 agent-centric mid-training 之后，把 RL 扩展到 code、general、visual 和 cyber 四类环境，并在一次 run 中混合不同任务与 agent harness。

## 单步 25K 条轨迹，数十亿训练 token

报告里最醒目的数字来自 RL batch：

- 1,568 个 prompts
- 每个 prompt 生成 16 条候选轨迹
- 每步约 25,000 条 sequences
- 每步 2.7B 到 3.7B training tokens
- 单条 sequence 平均约 110K 到 150K tokens
- 最大 context length 达到 1M

这类 workload 的主角是 fully asynchronous training。Rollout 长短差异巨大，多个 harness 的执行速度和外部环境也不一致。系统需要持续接收轨迹、稳定每类任务在 training batch 里的比例，并允许 partial rollout，避免少数长尾样本拖住整步训练。

在 MiMo-V2.6-Pro 的 RL 成本中，rollout 占 43.8%，training 占 43.5%，grader 占 12.7%。这个分布值得记住。Agentic RL 的 compute budget 几乎有一半花在生成经验，grader 也已经成为独立的系统组件。

随着 RL 成本增加，DeepSWE average@3 从 58.4 提升到 72.6（Pro），Flash 则从 48.7 提升到 65.7。报告为 Pro 和 Flash 的 RL post-training 分别投入约 260 万美元和 90 万美元。

## Grader 也要 scale

Binary tests 只能告诉模型任务是否通过，无法区分两个都 pass 的方案在质量、效率和行为上的差异。

MiMo 引入 groupwise agentic grading，在同一个 prompt 的候选组内比较不同轨迹。Groupwise Reward Synthesis 生成更细的质量信号，Groupwise Advantage Redistribution 再把相对差异写回训练 advantage。它可以奖励更短、更高效的解法，也能让长周期任务获得比 0/1 verifier 更密的信息。

这里有一个重要的工程判断：grader compute 并非纯 overhead。只要它能显著改善 credit assignment，额外的 12.7% 成本可能比继续扩大 rollout 更划算。真正的问题会变成 marginal return，下一单位 compute 应该投给更多 samples、更强 grader，还是更大的 policy update。

## 把 mixed-task agentic RL 做成系统

报告列出了一组容易被模型结构和 benchmark 分数遮住的基础设施：

1. **Unified trajectory representation**：统一表达不同 harness、工具调用、routing 信息和多模态 payload。
2. **Decoupled control and data planes**：控制面负责任务与调度，数据面缓冲和传输海量轨迹。
3. **Sample mixer**：在异步采样下维持各任务的 batch composition，配合 dynamic sampling 和 partial rollout。
4. **Training-inference consistency**：对齐 MoE routing 和 top-p candidate set，减少 rollout engine 与 training engine 的行为偏差。
5. **Router freezing**：在大规模 RL 中冻结 MoE router，降低路由漂移带来的不稳定性。

Harness diversity 也被当成训练维度。团队没有直接堆叠完整 production harness，而是从最小 agent loop 出发，拆分 system prompt、tools 和 context management，再为不同任务重组 mini-harness。这样可以控制变量，也减少生产系统中大量非任务约束对 credit assignment 的干扰。

## Reward hacking 需要多层防线

代码任务里最典型的漏洞是 solution leakage。Agent 可能从较新的 package、上游 GitHub 文件、残留 patch、build artifact 或网络中直接拿到答案，tests 依然会给出满分。

MiMo 的防线覆盖训练前后多个阶段：

- 在 mid-training 中加入错误行为与修正轨迹
- 清理 build log、verifier output、残留 patch、binary、bytecode 和 cache
- 截断 Git history，移除 base commit 之后的引用
- 使用 container-level network isolation
- 训练中做 adversarial screening
- 持续离线审计 agent trajectories

这部分和规模本身同样重要。RL 会放大任何稳定可利用的 evaluator 缺口，采样越多，找到捷径的机会越大。

## 对 post-training 团队的启发

MiMo-V2.6 最值得复用的内容是一套 scaling checklist：

- Policy 是否拥有足够丰富的 mid-training exploration space？
- Rollout、grader、training 三类 compute 如何分配？
- Mixed-task batch 在异步执行下能否维持稳定组成？
- Harness 差异是否带来可迁移能力，还是制造无奖励约束？
- Training engine 与 inference engine 是否保持采样和路由一致？
- Environment 中是否存在答案泄漏或 verifier shortcut？

团队还开放了 MiMo-V2.6-Distill-Qwen-9B、带 verifier 的多领域环境、端到端 RL framework 和可组合 mini-harness。很多设计可以先在 9B 规模验证，再判断哪些规律能延伸到 frontier run。

## 今天另外两篇相关论文

**TestPrism** 重新审视 coding agent 的 test generation。它用 300 个任务和 3,000 个候选实现评估 tests，要求测试套件接受全部有效实现，同时拒绝全部无效实现。14 种 agent 配置在 single-reference 评估上达到 59.67%，换成 Joint Success Function 后只剩 28.00%。生成 tests 一旦承担 verifier 或 RL reward 的角色，误杀正确实现会把错误 specification 写进训练信号。

**Cadence** 把 coding-agent monitor 做成动态闭环。轻微问题触发 advisory guidance，严重失控触发 replacement guidance；scheduler 再根据干预级别收紧或放松下一次检查。它在 300 个 SWE-bench Lite tasks 上，让 mini-swe-agent 和 Moatless 分别多解决 76 和 47 个任务，同时保持有竞争力的 token efficiency。

论文：<https://arxiv.org/abs/2610.11959>

另外两篇：<https://arxiv.org/abs/2610.12289>、<https://arxiv.org/abs/2610.12269>
