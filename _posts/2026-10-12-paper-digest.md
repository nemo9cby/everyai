---
title: "Paper Digest: 2026-10-12"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Agent Training, Software Engineering, Post-training]
---

今天最值得看的 paper 是 **ProgramGym: Scaling Training Environments for Long-Horizon Program Reproduction Agents**。

很多 coding benchmark 给 agent 一份 repository 和一个明确 issue，终点是修好 bug、通过 tests。ProgramGym 研究了一类更接近完整软件开发的任务：agent 先探索一个 reference program，自己弄清楚它做什么，再设计并实现一个行为一致的新程序。

这类任务的难点很容易被低估。需求没有完整写在 prompt 里，agent 需要通过交互发现 specification；实现周期更长，编译和测试失败会反复出现；训练环境还必须自动运行，并给出足够可靠的验证信号。

## 1,648 个可验证的程序复现环境

ProgramGym 包含两部分。Program-Env 负责把现有 repository 转换为训练环境，Program-Agent 负责执行长程开发任务。

Program-Env 用 feedback loop 反复检查并修复两类关键资产：

1. **Runtime sandbox**：程序能否在隔离环境中稳定构建和运行。
2. **Black-box tests**：测试能否通过外部行为判断复现结果，避免依赖原项目内部实现。

最终得到 1,648 个训练环境，覆盖 C、C++、Go 和 Rust。这个规模值得关注，因为 agent post-training 最稀缺的资产通常是可执行、可验证、彼此足够多样的任务环境。Prompt 可以快速生成，可靠 sandbox 和 verifier 需要大量工程。

Black-box evaluation 也改变了任务本身。Agent 无法靠复制内部结构满足测试，它需要识别真正影响外部行为的约束。对训练数据而言，这会产生更丰富的轨迹：探索 reference program、提出假设、设计模块、实现、编译、测试，再根据失败修订。

## Training data 和 agent harness 各自贡献多少

论文给出了一组很干净的结果。使用 ProgramGym 轨迹 fine-tune Qwen3.8-27B 后，ProgramBench 成绩从 34.87% 提升到 42.84%。说明环境生成出来的长程轨迹具有有效的训练价值。

Program-Agent 进一步把工作流拆成 analysis、implementation 和 review 三个隔离角色，并加入 revision management。每轮 revision 都可以执行 compilation check，失败后保留清晰的恢复点。这个 harness 将 base model 提升到 55.71%，将 SFT model 提升到 56.25%。

因此，ProgramGym 同时暴露了两个独立杠杆：

- 轨迹数据改善模型参数中的能力。
- 执行工作流改善推理时的规划、检查和恢复。

两者叠加后的差距并不大，base 和 SFT 只相差 0.54 个百分点。这可能意味着强 harness 吸收了相当一部分模型能力差异，也可能意味着当前 verifier 和 benchmark 已接近该工作流的瓶颈。论文值得进一步追问：哪些错误被 SFT 消除，哪些错误只能依靠 revision loop 处理？

## 为什么 program reproduction 适合 agent training

程序复现天然包含几个对 coding agent 很重要的训练信号。

首先，specification discovery 无法省略。Agent 必须主动调用程序、观察输出、覆盖边界条件，再形成对行为的解释。其次，任务要求系统级设计，局部 patch 很难完成。最后，编译器和 black-box tests 提供了便宜、可重复的反馈，适合大规模 trajectory collection，也适合后续 RL。

真正困难的部分会集中在 verifier quality。测试覆盖不足时，agent 可能实现一个只满足少数观察的近似程序。测试过度绑定具体输入时，训练又会奖励对 evaluator 的适配。Program-Env 的 test repair loop 因此很关键，它决定了环境规模能否转化为真实能力。

对 post-training 团队，ProgramGym 提供了一份实用 checklist：

- Sandbox build 是否稳定，依赖和 toolchain 是否可复现？
- Black-box tests 是否覆盖主要行为和边界条件？
- Reference program 是否会通过文件、日志或 package 泄漏实现？
- 不同语言和项目的 trajectory length 差异如何影响 batch scheduling？
- Sparse terminal reward 是否需要 process reward 或分阶段 verifier？
- Role isolation 带来的收益能否通过 training 蒸馏进单个 policy？

如果环境生产和 verifier repair 可以持续运行，这套框架很适合扩展为 coding-agent RL testbed。系统侧的核心工作会是并行调度 build、reference exploration、rollout、test execution 和失败重试，同时持续监测 leakage、reward hacking 和环境健康度。

## 今天另外两篇相关论文

**SoftwareGym** 把环境生产扩展到 computer-use agents。它自动合成 182 个可运行 GUI 应用，覆盖十二类软件，经过 GUI evaluation 和 iterative repair 后，平均任务通过率达到 91.5%。使用这些环境收集轨迹进行训练后，模型在严格 out-of-domain 的 OSWorld 上，Pass@3 从 41.39% 提升到 56.97%。这说明合成软件只要具备足够丰富的交互结构和可靠运行时，也能提供可迁移监督。

**Work2Skill** 关注人类经验如何变成 agent 可复用的技能。WorkRecorder 对齐 screen recording、键鼠事件和 accessibility tree，构建 350 条完整演示。Induction agent 再从证据中提取 workflow rules、step dependencies 和 completion checks。在四种执行模型上，技能注入都带来稳定增益。它给 agent memory 和 durable skill system 提供了一个很实用的评估方向。

论文：<https://arxiv.org/abs/2610.13395>

另外两篇：<https://arxiv.org/abs/2610.12611>、<https://arxiv.org/abs/2610.12984>
