---
title: "Paper Digest: 2026-09-21"
categories: [Paper Digest]
tags: [AI, LLM, Code Agents, Reinforcement Learning, GRPO, Software Engineering]
---

今天最值得看的 paper 是 **CodeMidas: Scaling Agentic Coding RL Environments from Code Itself**。

训练 coding agent 的 RL 数据有一个很现实的瓶颈：任务必须足够多、足够多样，还要有可靠 verifier。Issue、pull request 和 commit 可以提供天然任务，但只有留下完整开发痕迹的功能才能被利用，大量已经实现的代码因此沉睡在 repository 里。

CodeMidas 选择直接从 source code 生产 RL environment。输入一个可运行的 repository，agent 会探索其中已经实现的功能，写出 behavioral specification，基于原程序执行结果构造测试，再通过 execution check 和多次 solution rollout 验证、过滤任务。

最终数据集包含 5,545 个训练任务，来自 3,185 个 open-source repositories，覆盖 23 种编程语言和 15 个技术领域。

## 把 repository 变成 environment factory

CodeMidas 的 pipeline 可以拆成四步：

1. Agent 探索 codebase，寻找可独立描述和验证的功能。
2. 根据现有实现提炼 behavioral specification。
3. 运行原始代码，构造有 execution grounding 的 tests。
4. 用执行检查和 repeated solution rollouts 验证任务质量，过滤泄漏、歧义和过难或过易的样本。

这里最有价值的设计是把 agentic compute 用在 environment construction 的每个阶段。任务生成不再依赖单次 LLM 调用。Specification、tests 和可解性都要经受程序执行与多轮 rollout 的交叉检查。

对 coding-agent RL 来说，这相当于建立了一座自动化工厂：repository 是原料，最终产物是带 executable reward 的训练环境。

## GRPO 在五个 benchmark 上全部提升

作者使用生成的任务对 MiMo-V2.5 做 GRPO，五个不同类型的 benchmark 都有提升，其中包括：

- DeepSWE：+11.7%，测试真实 issue repair
- ProgramBench：+17%，测试 whole-program construction
- Terminal-Bench v2.1：+8.5%，测试 terminal work

这些结果的重要性在于 transfer。训练环境来自大量现有代码的功能重建，下游收益覆盖 repository repair、从零构建程序和 terminal 操作。Trajectory analysis 还显示，RL 后的 agent 会更深入地探索 codebase，并进行更多样的 self-verification。

Ablation 也给出一个朴素但关键的结论：高质量训练任务继续增加时，性能仍然上升。Coding-agent RL 的瓶颈很大一部分落在 verified task supply 上。

## 真正困难的是 verifier

CodeMidas 的扩展性很诱人，不过自动生成测试也会放大 verifier 风险。

一个测试可以稳定运行，却只覆盖 reference implementation 的偶然细节；它也可能泄漏答案、漏掉边界行为，或让 agent 通过 reward hacking 得分。Pipeline 因而需要持续监控：

- generated test 的 discriminativeness
- 多次 rollout 后的 pass-rate distribution
- specification 与 implementation 之间的信息泄漏
- 不同语言和 domain 的覆盖平衡
- environment generation cost 与下游收益
- targeted mutants 能否被测试拒绝

今天的第三篇论文 GameLogicBench 正好补充了最后一点。它用 tick-level assertions 检查游戏运行中的规则，并用“删除一项必要能力”的 mutants 验证 evaluator。如果 evaluator 连这些定向错误都抓不住，它也不适合成为 RL reward。

## 两种从代码中学习的路线

今天的第二篇 **Code2Skill** 也从 source code 出发，但产物是可检索的 procedural memory。它从 19,769 个活跃 GitHub repositories 中提取并验证了 1,006,822 条 skills，在 72 组 matched evaluations 中平均提升 11.7%。

这两篇 paper 放在一起看很有意思：

- CodeMidas 把代码转成 executable RL environments
- Code2Skill 把代码转成 grounded, retrievable skills

一个自然的下一步是组合两者。Verified skills 可以帮助 agent 更快完成 rollout，environment outcome 又能反过来验证、更新或淘汰 skills。Repository 由此同时提供知识与训练反馈。

CodeMidas 最值得复现的部分，是它对环境生产过程的抽象。只要 executable software 本身可以充当 oracle，历史 issue 和 commit 就不再是唯一入口。对于做 code-agent post-training 的团队，这可能比继续微调 policy architecture 更接近当前的数据瓶颈。

## 另外两篇

**Grounded Skill Synthesis from Code at Scale for Agentic Intelligence** 提出 Code2Skill，通过 source-body-blind reconstruction 与 source-aware comparison 验证代码中提取的 procedural knowledge，最终构建了超过一百万条记录的 CodeSkillBank。

**GameLogicBench: Evaluating Coding Agents on Runtime Game Logic with Tick-Level State Assertions** 包含 72 个 Godot gameplay tasks、403 个手工场景和 1,451 个 seeded test cases。20 种 model-scaffold 组合中，最好成绩为 52.78%；多数失败提交可以运行，却没有正确实现完整行为。

论文：<https://arxiv.org/abs/2609.22068>

另外两篇：<https://arxiv.org/abs/2609.05571>、<https://arxiv.org/abs/2609.21562>
