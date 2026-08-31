---
title: "Paper Digest: 2026-08-31"
categories: [Paper Digest]
tags: [AI, Coding Agents, Agent Evaluation, Post-Training, Reinforcement Learning]
---

今天最值得看的 paper，我会选 **LoopArena: Benchmarking Models as Runtime Controllers for Loop Engineering**。

长时间运行的 coding agent，真正困难的部分经常发生在每一轮 coding 之间。

当前实现做到什么程度？进度说明有没有过期？某个局部 test 通过，是否足以证明整个任务完成？下一轮应该继续写代码、补验证、调查失败，还是停止？

LoopArena 把这些判断从 coding Worker 身上拆出来，交给一个独立 Controller，再对 Controller 本身做评测。

这个设计让 agent orchestration 第一次成为一个相对干净的实验对象。

## 固定 Worker，只比较 Controller

每个 LoopArena run 都有三个角色。

**Worker** 是实际执行 coding task 的 agent。它可以读写 repository、调用工具、运行测试，并保留一条持续增长的工作对话。

**Reporter** 在每轮 Worker 交回控制权时临时启动。它读取 Worker history，并用 read-only tools 检查 workspace，随后生成四部分报告：task context、已经完成的工作、verification evidence 和 remaining issues。报告里的重要 claim 会引用对应的 Worker turn。

**Controller** 只能阅读 Reporter 生成的 Evidence Packet。它没有 coding tools，也不能直接查看 workspace。它输出一个结构化 Loop Contract，规定 Worker 下一轮要完成的 bounded assignment、需要提供什么证据，以及什么时候再次交回控制权。Controller 也可以决定停止并提交当前 workspace 给 evaluator。

比较不同 Controller 时，Worker、Reporter、coding tools、task environment、execution budget 和 evaluator 都保持固定。论文测试了 Qwen3.7-Plus、DeepSeek-V4-Flash-0731、GLM 5.2、GPT-5.5 和 Claude Opus 4.8。

很多 coding benchmark 测到的是 model、agent loop、tools 和 budget 的混合结果。LoopArena 的隔离让一个更具体的问题可测量：给定同一名 Worker，哪个 model 更擅长管理它？

## 三种评测成本

LoopArena 把 control ability 分成三个 execution scope。

**Type I** 只评估一次 control decision。Controller 看到一个 Evidence Packet，以及四个候选 Loop Contract。Benchmark 构建阶段已经逐个执行候选并用下游结果确定正确答案，所以评估新 Controller 时只需要一次 four-way choice，无需再次运行 Worker。

**Type II** 从完整任务的中间 checkpoint 开始，执行一个连贯的 task slice。Controller 与 Worker 会进行多轮真实交互，evaluator 也会检查截至该阶段的累计要求。

**Type III** 从原始 repository state 和完整 specification 开始，覆盖调查、实现、验证、恢复与最终停止。每个 Type II slice 都与一个 Type III full task 配对。

三层设计同时提供了便宜的 decision diagnosis、保留真实执行的短任务评测，以及完整的 long-horizon assessment。

## 完整任务成功率仍然低于 25%

Type II 和 Type III 共使用 27 个 official full tasks，每个 policy 对每个任务独立运行三次。成功要求 frozen evaluator 通过，同时遵守控制协议。

五个 Controller 在 Type III 的 Strict Success Rate 只有 16.05% 到 24.69%。最强结果也没有超过四分之一。

论文还加入两个 reference policy。

No control 让 Worker 接收原始任务后自行工作到结束。Fixed control 则模仿 persistent goal，每次 handoff 都机械地重述原始目标并让 Worker 继续。

Fixed control 在 Type II 把成功率从 39.51% 提高到 46.91%，可是在 Type III 中与 no control 同为 18.52%。短 task slice 里，持续提醒目标可能有用。完整任务会经历实现、验证、修复和停止等不同阶段，下一条指令必须根据新证据改变。

这也是 LoopArena 最有意思的结果之一。Goal persistence 可以防止遗忘，却不能替代 runtime judgment。

## 一个更便宜、同时保留排序的 proxy

完整 coding-agent evaluation 很贵。每个 Controller decision 都会触发 Reporter、Controller 和后续 Worker execution，Type III 还要从空白状态跑完整任务。

Type II 的 paired estimated inference cost 平均比 Type III 低 64.4%。在论文的 main Core criterion 下，五个 Controller 的 Type II 与 Type III 排序 Spearman correlation 达到 0.9747。九个在两边都有严格顺序的 Controller pair，没有一对发生 reversal，剩余一对包含 tie。

对做 agent post-training 或 model selection 的团队，这提供了一种实用结构：用 Type I 找 decision-level weakness，用 Type II 做快速 checkpoint iteration，定期再用 Type III 验证 long-horizon generalization。

这种 proxy 也有边界。论文发现换用不同 SCBench scoring rule 会改变绝对成功率与 Controller ordering。Type II 适合当前 task distribution 和 criterion 下的低成本比较，仍需用完整任务守住最终判断。

## Evidence Packet 与 Loop Contract 值得复用

LoopArena 的价值还在两个 interface。

Evidence Packet 把 coding progress 变成 read-only、带引用的状态摘要。Controller 得到的是 current state、已经存在的验证证据和 unresolved issues，而不是一段没有 provenance 的自由文本。

Loop Contract 把下一轮工作限制成一个明确 segment。它能表达继续实现、focused verification、recovery 或 stop，并规定交回控制权的条件。

这套接口适合用于生产系统的审计，也适合产生训练数据。一次 handoff 可以保存：

- repository checkpoint 与 Worker version
- Worker trajectory 和 Reporter citations
- Evidence Packet
- 多个候选 Loop Contract
- 每个 Contract 的 replay outcome
- evaluator receipt、token、cost 与 stop reason

有了 restorable checkpoint，同一个控制点可以并行执行多个候选 Contract。下游 executable outcome 可以生成 preference pair、process reward 或 offline RL sample。训练对象会聚焦于“下一轮该做什么”，credit horizon 也比完整任务的 terminal reward 更短。

## 对 distributed agent infrastructure 的启发

LoopArena 很适合拆成异构 pipeline。

Worker pool 负责昂贵的 repository execution。Reporter pool 做 read-only evidence extraction。Controller inference 相对轻量。Checkpoint service 保存可恢复 workspace，evaluator service 生成 machine-verifiable receipt。Type I 的候选执行可以在 benchmark construction 阶段并行完成，后续大量 Controller evaluation 只复用冻结的数据。

这里最需要守住的是 versioning。Policy checkpoint、Worker model、Reporter prompt、tool implementation、container image、evaluator、task snapshot 与 runtime limit 都会改变结果。缺少完整 run manifest，Controller gain 很容易与 Worker 或 harness update 混在一起。

论文也暴露了协议本身的 failure mode。部分 Controller 会耗尽 20,480-token output limit，或者输出无法解析的 Contract。长任务评测需要把 protocol compliance 当作真实能力，而不是在统计时悄悄重试掉。

LoopArena 最重要的判断很直接：coding agent 的 outer loop 需要独立评测。会写代码的 model，不一定会管理另一个会写代码的 agent。保持目标、理解证据、安排验证和选择停止，是一组仍然非常薄弱的能力。

## 另外两篇

第二篇是 **DART-SD: Diamond-topology Aware Retrieval and Tuning for Self-Distillation of Multi-Turn Tool-Calling Agents**。

它把 tool trajectory 表示成 Interaction-State Transition Graph。多个 order-independent subgoal 会让合法路径在相同 state 汇合，形成 diamond topology。DART-SD 找到失败 rollout 第一次离开 success-reachable region 的 Critical Topological Breakpoint，只训练后面的 recovery suffix，并保护此前正确 prefix。Qwen3-8B 的九项平均分从 base 的 29.60、SFT 的 41.64 提高到 45.58，最终成功轨迹也比数据构建时的 golden reference 更短。

第三篇是 **Rubric-to-Code Credit Assignment for Reinforcement Learning**。

RCCA 面向完整 HTML、CSS 和 JavaScript application，把用户可见的 functional rubrics 转成 code-localized RL signal。Hierarchical reward 先区分 format、source、runtime 和功能错误，evaluator attribution 再把每条 rubric 对齐到 event handler、state update、DOM fragment 或 CSS selector 对应的 tokens。Ling-RCCA-Flash 在 MiniAppBench 上比 base model 高 32.20 分，在 ArtifactsBench 上也超过 SFT model 4.48 分。

论文：<https://arxiv.org/abs/2608.28281>

另外两篇：<https://arxiv.org/abs/2608.18524>、<https://arxiv.org/abs/2608.27906>
