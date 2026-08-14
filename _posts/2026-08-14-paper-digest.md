---
title: "Paper Digest: 2026-08-14"
categories: [Paper Digest]
tags: [AI, Agents, Agent Harness, Code Agents, Skills, Reinforcement Learning]
---

今天最值得看的 paper，我会选 **DarwinX: Evolving Agent Harnesses Through Natural Selection**。

一个 LLM agent 的能力同时存在于两处：模型 weights，以及包围模型的 harness。Prompt、tools、skills、memory 和 control flow 都会决定它怎样行动。模型冻结之后，agent 仍然有很大的优化空间。

DarwinX 研究的就是这部分空间。它让多个 harness 形成 population，通过 benchmark verifier 做选择，保留能扩展能力且不破坏旧能力的变体。一次 evolution loop 在四个 benchmarks 上平均提高约 17 个点，Terminal-Bench 2.1 达到 84.7%，WebArena-Infinity real-task pass@1 从 43.5% 提高到 audit-clean 的 93.0%。在 Terminal-Bench 上进化的 harness 还能不经修改迁移到 SWE-bench Verified。

这篇论文最有价值的地方，是它认真处理了 agent self-improvement 的 regression 问题。

## Harness 也是 policy

Agent harness 通常包含 system prompt、tool schemas、skills、任务分解方式、verification loop、memory rules 和错误恢复策略。它们共同限制模型能看到什么、可以采取哪些动作，以及每一步会收到怎样的反馈。

所以，冻结模型不等于冻结 agent。修改一条 search rule、增加一个 verification step、重写一个 skill，都会改变 trajectory distribution。

常见的 self-improvement loop 采用 single lineage：运行任务，观察失败，编辑当前 harness，再运行下一轮。它实现简单，也容易发生 path dependence。某次修改解决了眼前失败，却破坏先前已经通过的任务。后续搜索又建立在这个变体上，回退与分支信息很快丢失。

DarwinX 把优化对象改成一组并存的 harness lineages。搜索可以保留多个方向，后续还可以 recombine，局部修改无需立即成为唯一的未来。

## Preserve-and-extend contract

DarwinX 的核心约束叫 **preserve-and-extend**。

候选变体想进入 population，需要保留现有 coverage，同时扩展新 coverage。一个修改即使修好了当前 failure，只要让旧任务退化，就不会被直接接受。

这个规则很像 production software 的 regression suite。Agent optimization 追求更高 aggregate score 时，平均数可能掩盖具体能力的丢失。Preserve-and-extend 用 task-level behavior 约束搜索方向，让每轮 evaluation 同时承担 learning signal 与 regression gate 两个职责。

这种设计也减少了 cherry-picking。Fitness 来自 benchmark 自带 verifier，不需要 gold solution，也不靠人挑选看起来最合理的修改。失败、teacher evidence 和 agent 自己总结的 evidence 最终进入同一套 harness-editing interface。

## Archive 为什么重要

Population search 的另一个关键是 archive。

不同 harness 可能覆盖不同任务。一条 lineage 善于 terminal debugging，另一条更擅长 web navigation。只保留当前 aggregate score 最高的版本，会过早丢掉具有互补价值的策略。

Archive 保存这些替代路径，让后续 mutation 和 recombination 可以重新使用。它保留的是 behavioral diversity，也保留了搜索过程中的 option value。

这和 post-training 里的 rollout diversity 很接近。多样性只有进入选择与组合机制，才会变成可积累的能力。单纯生成更多 variants，随后用一个 scalar score 全部压平，通常会迅速收敛到局部模式。

## 四组 benchmark 结果

论文逐步拉开 evolution signal 与最终测试之间的距离，用来检查 harness 是否只记住 benchmark-specific patches。

- Terminal-Bench 2.1 在 matched base model 上提高 7.7 点，达到 83.2%
- 使用更强 base model 时达到 verified frontier 的 84.7%
- TerminalWorld held-out split 达到 68.3%，超过所有对比的 off-the-shelf agents
- WebArena-Infinity real-task pass@1 从 43.5% 提高到 audit-clean 的 93.0%
- Terminal-Bench 2.1 上得到的 harness 不经修改迁移到 SWE-bench Verified

最后一项尤其关键。若改进主要来自 task-specific patch，它很难跨任务、verifier 和 base model 保持效果。Unchanged transfer 说明 harness 捕获了更一般的 agent operating procedure，至少在这些测试中如此。

## Evaluation compute 变成持久能力

普通 benchmark evaluation 在结束后只留下一个 score。DarwinX 让 evaluation 结果反过来修改 harness，于是同一笔 compute 还能产生可复用 artifact。

这个思路对 code agents 很现实。大量 SWE-bench 或 terminal rollouts 已经包含丰富 failure evidence：搜索过早停止、没有验证 patch、误读 test output、修改范围失控、依赖安装失败后没有恢复。把这些失败压缩成 executor 可以稳定遵循的 skill 或 control-flow rule，下一批任务就能复用。

当然，evaluation compute 也会迅速膨胀。Population size、mutation count、recombination frequency 和 regression suite coverage 都会增加成本。真正的 systems problem 是如何分配 budget：哪些失败值得产生新 lineage，哪些旧任务需要每轮回归，哪些 harness 改进已经足够稳定，可以 distill 进 SFT 或 RL trajectories。

## 对 post-training 的启发

DarwinX 可以看作发生在 weight space 外部的 policy search。

Verifier 提供 reward，harness variant 对应 policy configuration，archive 保留多样性，preserve-and-extend 构成行为约束。它和 weight-level RL 的边界也很有意思：高频、稳定、跨任务的 harness patterns 可能适合蒸馏进模型；仍在快速变化、依赖特定工具的规则更适合保留在外部。

一个值得做的 hybrid loop 是：

1. 先用 population-based harness evolution 找到可靠 operating procedures
2. 把成功与失败 trajectories 整理成 SFT 或 RL data
3. 训练后的模型重新进入 harness evolution
4. 比较改进来自 weights、harness，还是两者的 interaction

这能回答一个很实际的问题：什么时候继续增加 inference-time scaffold，什么时候应该把能力训练回模型。

## 应该怎样解读

高 benchmark gains 仍需关注评估独立性。Preserve-and-extend contract 依赖 verifier 和 regression set，二者覆盖不到的行为可能继续退化。WebArena 一类环境还存在状态变化、任务污染与审计口径问题，论文报告 audit-clean 结果是必要步骤，长期复现依然重要。

Population search 也可能产生越来越复杂的 harness。若缺少 complexity penalty、可读性检查和定期 consolidation，积累的 rules 会互相冲突，并增加 context 与维护成本。

因此，下一步除了追求更高 pass rate，还应报告 harness size、inference cost、跨模型 transfer、规则冲突率，以及在工具版本变化后的 robustness。

## 为什么今天选它

DarwinX 给 agent self-improvement 加上了三个很扎实的部件：population、archive 和 regression contract。

它把 prompt、tools、skills 与 control flow 当作可验证、可组合、可积累的系统资产。模型 weights 保持冻结，agent capability 仍能通过 evaluation 持续增长。

对正在构建 coding agents、skills platform 或 post-training pipeline 的团队，这条路线很值得认真试验。训练模型很贵，harness evolution 的成本也不会消失，但它提供了一个更快、更透明，也更容易回滚的能力搜索层。

## 另外两篇

第二篇是 **Intern-S2-Preview: Scientific Agentic Foundation Model**。

它给出一套规模很大的 scientific agentic model stack：multimodal scientific pretraining，随后进行 SFT、scalable multi-task RL、black-box 与 white-box agentic RL，以及 on-policy distillation。更值得工程团队看的，是 partial rollout with off-policy correction、adaptive length regularization、online speculative decoding、robust multi-task optimization 和 trace-aware experience assembly。这些细节直接影响长轨迹训练的 throughput、stability 与 actor-learner lag。

第三篇是 **SKILLER: Language-Level Reinforcement Learning for Reusable Skill Extraction in Small Language Models**。

SKILLER 让强模型担任 actor 和 critic，把 small-model agent system 当作 environment，通过自然语言 reward 与 feedback 自动产生 executor-specific skills。Qwen3.5-9B 和 4B 在五个 benchmarks 上取得明显提升，单 skill tasks 上可以匹配强闭源模型。它对 OpenClaw、Codex 一类 skill systems 的启发很直接：skill 应当针对 executor 和 verifier 优化，不能只看文档写得是否完整。

论文：<https://arxiv.org/abs/2608.07545>

另外两篇：<https://arxiv.org/abs/2608.13505>、<https://arxiv.org/abs/2608.10538>
