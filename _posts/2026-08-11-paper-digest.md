---
title: "Paper Digest: 2026-08-11"
categories: [Paper Digest]
tags: [AI, Code Agents, Software Engineering, Benchmarks, Refactoring, Post-Training]
---

今天最值得看的 paper，我会选 **SWE-Bench ProMax: Benchmarking Agents on Large-Scale Multilingual Code Refactoring**。

它选了一个 code agents 还远没有解决、同时非常接近真实工程的能力：large-scale refactoring。模型需要在多个文件之间协调修改，保持外部行为一致，并让完整 test suite 继续通过。

SWE-Bench ProMax 收录 170 个来自真实 commits 的任务，覆盖 Python、Java、TypeScript、Go、C、C++ 和 Rust。任务平均涉及 11.4 个文件和 261.6 行代码变更。最强 frontier model 在两个 agent scaffolds 下的最佳 resolve rate 只有 41.2%。

## 为什么需要新的 SWE-bench

现有 code-agent benchmarks 正面临两个问题。

第一是 saturation。随着模型和 harness 共同进步，一些 benchmark 已经很难继续区分 frontier systems。

第二是 evaluation quality。论文引用的一项近期 audit 发现，SWE-bench Verified 中接近 60% 的未解决实例包含有问题的 tests。有些 tests 过窄，会拒绝合理实现；有些 tests 又检查 issue description 没有提出的要求。训练数据污染也会让模型直接复现 gold patch，进一步削弱分数的解释力。

Refactoring 提供了更硬的能力探针。一个 agent 需要理解 repository structure、追踪 cross-file dependencies、规划修改顺序、处理编译或类型错误，并通过 regression tests 验证行为。任务通常存在多种正确实现，评测设计必须允许合理的等价解。

## ProMax 怎样做 benchmark curation

SWE-Bench ProMax 的核心价值有一半来自 task difficulty，另一半来自人工 curation。

作者从真实 refactoring commits 构建任务，然后重写 issue descriptions，给 agent 提供明确且独立的 specification。Test suites 经过人工审查，移除只接受某一种实现的 overly narrow tests，也移除检查 unstated requirements 的 overly broad tests。

团队还过滤掉修改范围太小、cross-file coordination 不足的实例。留下来的任务平均修改 11.4 个文件，规模明显超过常见的局部 bug fix。七种语言的覆盖则让 benchmark 可以测量 agent 能力对 language ecosystem、build system 和 tooling 的敏感度。

这种做法很重要。Code-agent evaluation 的瓶颈越来越多地出现在 benchmark 本身。测试是否准确、spec 是否完整、gold patch 是否泄漏，都会改变最终 resolve rate 的含义。

## 41.2% 说明了什么

Frontier models 在两个 agent scaffolds 下评测，最佳结果为 41.2%。这个数字留下了足够的 headroom，也说明 repository-scale refactoring 仍然需要更好的 planning、context management 和 verification。

对 agent team 来说，aggregate resolve rate 只是第一层结果。更值得继续分析的是：

- agent 能否在修改前识别完整的 dependency surface
- 失败主要来自漏改文件、错误 API migration，还是 regression handling
- 不同 scaffold 如何影响搜索、测试和 recovery behavior
- Python 与 Rust、C++ 等语言之间的差距有多少来自模型，有多少来自 toolchain
- partial progress 能否形成稳定的 SFT targets 或 process rewards

这些 failure traces 可以直接进入 post-training pipeline。成功轨迹适合用于 repository-scale SFT；可验证的中间状态可以支持 process supervision；测试失败与修复循环则为 agentic RL 提供天然反馈。

## 对 code-agent 系统的启发

第一，evaluation matrix 应该同时覆盖多个 languages 和 scaffolds。单一 harness 下的高分容易混入 context assembly、retry policy 和 tool design 的帮助。

第二，训练数据应该增加 large-scale refactoring trajectories。局部 patch 数据能训练编辑能力，很难充分覆盖跨文件 planning、接口迁移和 regression control。

第三，benchmark maintenance 需要持续投入。随着模型记住公开 patches，静态 benchmark 的有效期会缩短。重写 specifications、人工审查 tests、跟踪 contamination，都会成为长期基础设施。

第四，系统优化不应只盯着最终 pass/fail。对于平均修改 11 个文件的任务，定位 agent 在哪一步丢失 dependency、引入 regression 或停止验证，通常比一个标量分数更有训练价值。

## 为什么今天选它

SWE-Bench ProMax 把 code-agent evaluation 推到了一个更有工程含量的区域。任务足够长、足够跨文件、覆盖多种语言，同时对 specification 和 tests 做了强人工质量控制。

它也给 post-training 留出了清晰入口。我们可以收集多 scaffold trajectories，研究语言间的 failure modes，把 verified partial progress 变成训练信号，再观察模型是否真正学会 repository-level planning。

## 另外两篇

第二篇是 **Macaron-V1: Towards Open Continual Learning with Self-Improvement and Mixture-of-LoRA**。

Macaron 把 continual learning 建模为 versioned model-harness pairs 的递归改进。它冻结 base model，用 Mixture-of-LoRA 组合 chat、agent、coding 和 GenUI specialists，并在每个 user turn 选择一个 adapter。整个技术报告还覆盖 agentic RL、long-context RL、post-training infrastructure 与 sparse model stability。最值得验证的是 adapter routing、capability interference，以及多轮部署后 gains 能否持续累积。

第三篇是 **Ouroboros: A Self-Developing Frontier Coding Agent with Reviewed Core Evolution**。

Ouroboros 让 agent 的 tools、prompts、context assembly 和 core implementation 通过 reviewed commits 持续改进。Benchmark 使用 frozen snapshots，长期部署则沿独立 lineage 继续演化。这个设计把 self-improvement 落到软件版本管理、review、reproducibility 和 safety boundaries 上，对 persistent coding agents 很有参考价值。

论文：<https://arxiv.org/abs/2608.09802>

另外两篇：<https://arxiv.org/abs/2608.09819>、<https://arxiv.org/abs/2608.08311>
