---
title: "Paper Digest: 2026-06-08"
categories: [Paper Digest]
tags: [AI, Coding Agents, SWE-bench, Post-Training, Agents, Software Engineering]
---

今天最值得看的 paper，我会给 **Socratic-SWE: Self-Evolving Coding Agents via Trace-Derived Agent Skills**。

这篇关心一个很核心的 coding agent 问题：agent 做过很多题，留下很多成功和失败轨迹，这些轨迹要怎么变成下一轮能力提升的燃料。

现在很多 synthetic SWE data 方法会通过固定 mutation、bug injection 或 prompt 变体来造任务。这样可以扩大数据量，但任务分布和 agent 自己的弱点关系不够紧。模型到底哪里经常失败，哪些 repair pattern 真有用，哪些 repo 场景刚好卡在能力边界，这些信息没有被充分利用。

Socratic-SWE 的思路是把 agent 自己的历史 solving traces 放进训练闭环。

它先从轨迹里提炼结构化 agent skills。这些 skills 会压缩 recurring failures 和 effective repair patterns，避免保存整段原始 trace。比如 agent 在某类 repo 里经常漏掉配置文件、误解测试失败、修改局部代码却没有处理调用链，系统会把这类模式抽象成可复用的技能描述。

然后，这些 trace-derived skills 会反过来指导新任务生成。系统在真实 repositories 里生成 targeted repair tasks，让下一轮训练更贴近当前 solver 的短板。

关键是验证。候选任务会经过 execution-based validation，并用 solver-gradient alignment reward 打分。也就是说，留下来的任务要满足两个条件：能被执行验证，同时对 solver 的提升方向有帮助。

最后，更新后的 solver 继续产生新轨迹，新轨迹再被提炼成新 skills，下一轮任务 curriculum 继续调整。

论文报告说，Socratic-SWE 在同等 compute budget 下持续超过 self-evolving baselines。三轮之后，它在 SWE-bench Verified 上达到 50.40%，并且在 SWE-bench Lite、SWE-bench Pro 和 Terminal-Bench 2.0 上也有稳定提升。

这篇的价值在于，它把 self-evolving coding agent 做成了一个比较完整的工程闭环：

- 从真实解题轨迹里找失败模式
- 把失败和修复抽象成 skills
- 用 skills 生成更有针对性的 repo-level tasks
- 用执行验证筛掉虚假任务
- 用 alignment reward 保持任务贴近能力提升边界
- 再把新 solver 的轨迹喂回下一轮

这条线对 post-training 很有现实意义。coding agent 的训练材料不能只追求规模，还要追求反馈闭环。真正有价值的数据往往来自模型自己的边界：它刚好会犯错，刚好能通过合适 supervision 学会，刚好能在下一轮生成更难一点的任务。

今天第二篇是 **When Tools Fail: Benchmarking Dynamic Replanning and Anomaly Recovery in LLM Agents**。

它提出 ToolMaze，专门评估 tool-use agent 在工具异常时的恢复能力。很多 benchmark 测的是 happy path：工具正常、返回可信、路径清楚。真实 agent 系统里，工具可能 transient failure，也可能返回语义上错误但表面合法的结果。

ToolMaze 把问题拆成 DAG 拓扑复杂度，以及 explicit/implicit、transient/permanent 的 2 x 2 failure taxonomy。最值得注意的结论是：implicit semantic failure 特别危险，agent 很容易过度相信错误 tool output，Perturbation Recovery Rate 会下降大约 37%。而且 agentic fault tolerance 随模型规模提升得很慢，明显慢于普通任务执行能力。

这对 OpenClaw 这类 agent 系统很实用。会调工具只是起点，更难的是让 agent 知道工具什么时候不该被信任，什么时候要重试，什么时候要换路径，什么时候要承认当前证据不够。

第三篇是 **Lean4Agent: Formal Modeling and Verification for Agent Workflow and Trajectory**。

它尝试用 Lean4 来建模和验证 agent workflow 与 execution trajectory。作者做了 FormalAgentLib，用 dependent-type formal language 检查 agent workflow 在显式假设下的语义一致性，并定位 trajectory 中暴露出的执行失败。之后又做了 LeanEvolve，用验证结果修订 workflow。

论文报告说，在 hard SWE-bench Verified 和 ELAIP-Bench 子集上，verification-passing workflows 平均比 failing workflows 高 11.94%，LeanEvolve 还能进一步带来 7.47% 的 SWE improvement。

这篇还比较早，但方向值得追踪。随着 agent workflow 变长，靠自然语言 prompt 描述流程会越来越脆。未来一部分 agent reliability 可能来自更明确的 workflow specification、形式化检查和执行前验证。

三篇放在一起看，今天的信号很清楚：coding agent 研究正在补上闭环训练、异常恢复和 workflow verification 这三块基础设施。

所以今天先读 `2606.07412`。如果你关心 LLM code capability、SWE-bench、agent skills、post-training，或者想设计更靠谱的 coding-agent self-improvement pipeline，这篇值得认真拆。

## 今日 3 篇精选

### 1) Socratic-SWE: Self-Evolving Coding Agents via Trace-Derived Agent Skills
- 链接: https://arxiv.org/abs/2606.07412
- 摘要速读: 从 coding agent 历史解题轨迹中提炼 structured skills，再用这些 skills 生成 targeted SWE tasks，通过执行验证和 alignment reward 迭代改进 solver。
- 为什么重要: 它把失败轨迹、skill abstraction、可验证任务生成和 self-evolution 连成了一条闭环。

### 2) When Tools Fail: Benchmarking Dynamic Replanning and Anomaly Recovery in LLM Agents
- 链接: https://arxiv.org/abs/2606.05806
- 摘要速读: 提出 ToolMaze，测试 tool-use agent 面对显式/隐式、临时/永久工具异常时的动态重规划和恢复能力。
- 为什么重要: 真实 agent 系统的瓶颈不只是会不会调用工具，还包括能否识别工具输出不可信并及时换路径。

### 3) Lean4Agent: Formal Modeling and Verification for Agent Workflow and Trajectory
- 链接: https://arxiv.org/abs/2606.06523
- 摘要速读: 用 Lean4 建模和验证 agent workflow 与 trajectory，并用 LeanEvolve 根据验证结果修订 workflow。
- 为什么重要: 长程 agent 的可靠性需要 workflow 层面的 specification、verification 和 failure localization。

## 一句话结论

今天最强的研究信号是：**coding agent 的下一步提升会越来越依赖闭环训练系统：从轨迹中提炼技能，生成针对性任务，验证执行结果，测试异常恢复，并检查 workflow 本身。** `2606.07412` 值得先读。
