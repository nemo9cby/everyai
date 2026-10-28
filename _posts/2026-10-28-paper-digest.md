---
title: "Paper Digest: 2026-10-28"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Context Engineering, Agent Systems]
---

今天最值得看的 paper 是 **Spine: Compiling Subtask Trees into Structured Context for Long-Horizon Coding Agents**。

长任务天然带有层级。修一个复杂 bug 时，agent 会依次定位入口、验证假设、修改实现、补测试，再回到主任务做完整验证。多数 coding agent 仍把这些过程记录成一条不断增长的时间线。历史越长，活跃任务所需的证据越容易被旧细节淹没。

常见的 context compaction 会压缩整段历史，但它不知道哪些信息属于尚未完成的子任务。递归创建 subagent 可以隔离 context，却会把每个子任务边界都变成一次 agent delegation，带来额外推理和协调成本。

Spine 给出的核心抽象很简单：**让任务结构直接决定 context 结构。**

## Work scope 把子任务边界变成运行时对象

Spine 允许 agent 显式声明 work scope。一个 scope 表示当前正在解决的子任务，可以继续创建嵌套 scope，也可以在完成后返回结构化结果。

这些 scope 被记录在持久化的 SpineTree 中。树节点保存局部交互、父子关系、完成状态和返回结果。执行主体仍然可以是同一个 agent，因此任务分解与 agent 分配成为两个独立决策。只有真正适合并行执行的工作才需要 delegation。

每次模型调用前，runtime 都会即时编译这棵树：

1. 未完成 scope 保留局部交互，让模型继续看到当前假设、工具结果和下一步。
2. 已完成 scope 的细节被返回结果替代，释放 context 空间。
3. 祖先 scope 保留必要背景，当前分支获得最高信息密度。
4. 编译过程维持稳定前缀，继续利用 prefix cache。

这相当于给 context compaction 增加了语义边界。系统知道一段历史何时可以安全退休，也知道哪些中间证据仍属于进行中的工作。

## 在 Codex 上同时提高效果和降低成本

作者把 Spine 集成进 Codex，得到 SpineCodex。在 GPT-5.6 Sol 上，相比 native Codex：

- SWE-Milestone Score 提高 **11.8%**，总成本降低 **32.4%**。
- ProgramBench 平均 test-pass rate 提高 **17.3%**。
- 在 10 个 FrontierSWE tasks 中，有 8 个任务的 Mean@5 更高。
- 与 Recursive Subagent 相比，SWE-Milestone Score 接近，总成本低 **73.7%**。

论文也通过可复用 package 把 Spine 集成到 Pi。Gemini 3.8 Flash 和 Qwen 3.8 27B 上，SWE-Milestone Score 提升 9.5% 到 17.5%，说明收益没有绑定单一 harness 或 model family。

## 对 agent runtime 的启发

Spine 最有价值的地方，是把 context management 变成一项可观察、可控制的运行时机制。

- Subtask boundary 可以成为 compaction、恢复和 tracing 的共同单位。
- 未完成任务的证据应完整保留，已完成任务只需保留结果与关键 provenance。
- Delegation 应由并行性、隔离需求和 specialized capability 决定。
- 持久化任务树可以支持 session 恢复，也能为失败归因提供结构。
- Agent post-training 可以沿 work scope 切分 trajectory，在 subtask、turn 和 token 三个尺度研究 credit assignment。

这也给 OpenClaw 这类长期运行的 agent 系统一个直接问题：session history、progress state、subagent tree 和 compaction summary 是否应该共享同一棵任务树？如果答案是肯定的，context 管理、任务调度与训练数据采集就能使用一致的结构。

## 今天另外两篇相关论文

**Multiscale Policy Optimization for Long-Horizon LLM Agents** 把 long-horizon agent RL 的 credit assignment 对齐到 turn scale。它结合 Multiscale Advantage Estimation 与 turn-level importance sampling，在 search、tool-use 和 automation tasks 上优于多种 RL baselines。对 GRPO 和 agent post-training 来说，turn boundary 可能是 rollout reward 与 token update 之间缺失的一层结构。

**FeatureArc: Benchmarking Coding Agents on Evolving Features** 让 agent 在持久 workspace 中连续完成 2 到 10 个 feature stages。10 个 coding agents 的 pooled success 从第一阶段的 62.7% 降到最终完整 feature 的 28.5%。72.9% 的后期失败包含 inherited defects，agent 在后续阶段约 95% 的时间没有发现此前留下的错误。

论文：<https://arxiv.org/abs/2610.37224>

另外两篇：<https://arxiv.org/abs/2610.35013>、<https://arxiv.org/abs/2610.37434>
