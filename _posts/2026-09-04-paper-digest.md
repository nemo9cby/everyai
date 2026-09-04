---
title: "Paper Digest: 2026-09-04"
categories: [Paper Digest]
tags: [AI, Coding Agents, Post-Training, SFT, Agent Environments, Distillation]
---

今天最值得看的 paper，我会选 **Terminal-Universe: Turning Agent Trajectories into Scalable Terminal Environments**。

训练 code agent 时，trajectory 已经积累了很多，真正稀缺的是可以反复执行、产生反馈、继续提出新任务的 environment。一条 trajectory 只记录一次 interaction；一个 executable workspace 可以重新查询，生成多道带 verifier 的任务，也可以继续 rollout。

Terminal-Universe 的核心想法，是利用 trajectory 里的 tool-execution history 反向恢复当时的 workspace。系统重放文件操作，把 agent 修改过的文件恢复到修改前状态，再由 completion agent 补齐缺失文件和 dependency。拿到可运行的 partial workspace 后，系统会重建原始任务，也会在同一个环境中合成新任务。

作者将规模扩展分成两个维度：

- Breadth：挖掘相关 environment 之间的依赖关系，生成跨多个 codebase 的 query。
- Depth：把 single-turn task 延展为 multi-round session，由 user agent 提供反馈和需求修订。

最终，Terminal-Universe 从公开 terminal-agent trajectories 中构建出 37.3k 个 task-sufficient environment。用这些数据对 Qwen3.5-27B 做 SFT 后，Terminal-Bench 2.1 的 single-round 表现提升 11.9 分，EvoCode-Bench v2 的 multi-round MT@4 提升 13.8 分。

## 为什么 environment 比 trajectory 更值钱

Trajectory 是一次已经结束的 demonstration。它包含当时的 prompt、工具调用和结果，却很难改变任务、重新执行或获得新的 verifier feedback。

Executable environment 是一项可持续生产数据的资产。同一个 workspace 可以支持：

1. 重建原任务，形成新的 rollout。
2. 修改需求，合成难度不同的新任务。
3. 运行测试和命令，提供可验证 reward。
4. 加入多轮用户反馈，训练 requirement refinement。
5. 与相关 workspace 组合，覆盖真实开发中的跨仓库操作。

这个视角对 post-training 很重要。继续收集更多静态 trace 的边际价值会下降，而恢复一个可运行环境，相当于获得一个小型 task generator 和 evaluator。

## Pipeline 怎么工作

第一步是 trajectory replay。系统读取文件创建、修改、移动和删除记录，逆向恢复每个文件在 agent 操作前的状态。

第二步是 workspace completion。公开 trajectory 往往缺少未被访问的文件、dependency 或环境配置，completion agent 负责补齐这些部分，使 partial workspace 达到 task-sufficient。

第三步是 task reconstruction。系统根据原始 interaction 恢复用户意图，并在重建 workspace 上验证任务仍然成立。

第四步是 task synthesis。系统继续探索 workspace，生成新的 single-repo、cross-workspace 与 multi-round task。

真正需要审计的环节集中在 workspace completion。补齐过程可能引入原始环境中不存在的信息，或者把本来困难的任务意外简化。一个可靠的数据系统需要保存 source trajectory、文件恢复记录、补齐 diff、dependency snapshot、测试结果与 task provenance，让训练收益可以追溯。

## 对 code-agent post-training 的启发

这篇论文提供了一条很实用的数据飞轮：

`trajectory → recovered environment → new tasks → new rollouts → stronger agent`

其中，workspace reconstruction 和 validation 天然适合分布式执行。每个环境可以作为隔离 job，由 Ray 调度 dependency installation、test execution、task synthesis 和 rollout。失败环境可以按 error type 分类，避免静默进入训练集。

下一步最值得做的实验，是在同一批 recovered environments 上固定 execution budget，分别比较 SFT、offline preference optimization 和 online RL。这样才能看清收益来自 demonstration density、verifiable feedback，还是 multi-round curriculum。

## 另外两篇

第二篇是 **Compile by Training: Turning Natural-Language Specifications into Local Neural Functions**。

它把自然语言 specification 编译成一个可复用的 neural function。Teacher model 在 compile time 生成 task-specific examples，再训练 compact interpreter 的小 adapter。部署时不再依赖 teacher，adapter 可以像软件一样存储、版本化和组合。在 FuzzyBench-Hard 上，它达到 83.6% semantic accuracy，代价是大约一分钟的 compile time。

第三篇是 **Rethinking On-Policy Distillation of Large Language Models II: One Training Example**。

作者发现，一条 query 已经能覆盖 full-data OPD 访问状态的 71.5%，16 条语义不同的 query 达到 98.9% state coverage，并匹配 full-data training。Rollout 很快暴露出广泛 supervision，student 对 teacher 的吸收仍需要数百步。这个结果提示 OPD 的主要瓶颈可能在 step efficiency，而非继续堆训练 query。

论文：<https://arxiv.org/abs/2609.04148>

另外两篇：<https://arxiv.org/abs/2609.04199>、<https://arxiv.org/abs/2609.04172>
