---
title: "Paper Digest: 2026-09-17"
categories: [Paper Digest]
tags: [AI, LLM, Code Agents, SFT, Reinforcement Learning, AI for Science]
---

今天最值得看的 paper，我会选 **ScienceIDE: Turning World's Scientific Codebase into Agent Learnable Environments**。

科学计算软件里沉淀了几十年的知识。天体物理、海洋气候、等离子体、材料、量子和粒子探测器的代码，包含可运行的模型、数值方法与领域约束。直接把这些 repository 交给 agent，通常得不到可靠的训练数据。

原因很实际。代码可能依赖 Fortran、MPI、特定编译器和复杂 toolchain；正确性也很少等于 unit test 全绿。两个输出存在微小数值差异，可能仍然表达相同物理结果。数组顺序、并行分块和 eigenvector phase 可以变化，守恒量、积分范数和统计分布却必须落在合理范围内。

ScienceIDE 解决的核心问题，是把这种隐含的 scientific contract 编译成可执行环境，让 evaluation、SFT 和 RL 共用同一套可信检查。

## 从 codebase 到可训练环境

ScienceIDE 从一个 pinned upstream revision 开始。Agent 负责调查依赖、build assumptions、official tests 和 examples，domain expert 决定模块边界、科学 observable 与等价条件。

每个 environment 包含四类关键资产：

- 可编辑的 agent workspace
- 固定 runtime 与 source provenance
- 一组经过校准的 scientific checks
- 与 workspace 隔离的 private verifier

Check 的单位是固定输入、graded output 和 pass policy。对于稳定输出，系统使用 pointwise tolerance：候选值与参考值的差异必须小于 absolute tolerance 与 relative tolerance 的组合。对于随机流、快速发散系统、sampling statistics 或离散输出，系统比较 moments、distributions、conserved values 或 integral norms。

这里有一个很重要的工程判断：检查的是物理 observable，不检查 bookkeeping。粒子属性要通过 particle ID 对齐，不能依赖数组位置；timing、rank layout、chunk layout、random draw 和 adaptive step count 也不会被误当成科学结果。

每条 check 还要经过 nominal input、variant input 和可选 alternate build 的校准。Domain expert 最终确认 tolerance、window 与科学依据。Task author 之后可以生成修复、实现、复现、集成或加速任务，acceptance contract 始终保持稳定。

## 一套 contract 服务三种用途

Environment-specific factory 会把通用操作与本地代码知识结合起来，提出 mutation、缺失实现和其他候选任务。候选必须在最终 grading environment 中通过 executable validation，才会进入 task bank。

以 repair task 为例，known-valid witness 必须通过，带缺陷的 baseline 必须真实偏离 scientific checks，reference repair 必须消除这个差异。Harness error 会单独记录，不会被算成 agent failure。

Agent 执行任务时，系统保存 tool actions、observations、submitted artifacts、check-level rewards、execution status 和 resource use。相同 episode interface 可以支持三种消费方式：

- Evaluation 固定 task 与 interaction budget
- SFT 选择通过 verifier 的 trajectories，监督文本和 tool calls
- RL 直接读取 private verifier 产生的在线 reward

这套分层很适合长期维护。模型、trainer 和 harness 可以替换，科学内容与 acceptance criteria 仍然可以复用。

## 当前规模与 benchmark 结果

ScienceIDE 目前包含 27 个 scientific codebases、64 个 environments、2,812 个 tasks 和 1,076 个 executable checks。任务主要集中在 repair 与 implementation，分别有 2,515 和 295 个；acceleration 目前只有两个。

ScienceIDE-Hard 选择了 85 个任务，来自 PLUTO、Athena++、MITgcm、LAPS 和 PHANTOM 的 18 个 environments。所有 agent 获得相同 container、instructions 与一小时 budget，成功条件是输出满足 private scientific reference，而非代码能够运行。

15 个 model-harness systems 中，Claude Fable 5.1 达到 67.1%，Claude Opus 5 为 64.6%，GPT-6 Astra 为 63.1%。结果也揭示了明显的 cost-quality trade-off。Astra 平均 9.4 分钟、13.9k output tokens、约 3.56 美元完成一项任务；Fable 平均 16.8 分钟、85.7k tokens、约 7.90 美元。

时间预算会改变排名。十分钟时 Astra 已达到 49.6%，Fable 只有 25.9%；Fable 大约在 31 分钟后反超。对于 agent infrastructure，这说明最终成功率不足以描述系统，time-to-solution、token volume、execution cost 和 budget exhaustion 都要进入评估记录。

## Verified trajectories 用于 SFT

作者使用 GPT-5.6-sol 生成 demonstrations，再通过 numerical-equivalence verifier 筛选。训练集包含 564 个 tasks 的 4,567 个 trajectory segments，validation 包含 81 个 tasks 的 544 个 segments。Qwen3.5-4B、Qwen3.5-9B 和 Qwen2.5-72B-Instruct 使用 LoRA 训练三个 epochs。

在 held-out scientific repair 上，4B 模型在 PLUTO-Particles-Dust 的 mean repair reward 从 0 提高到 0.3333。9B 模型在 PLUTO-RMHD/ResRMHD 从 0 提高到 0.2857，在 LAPS 从 0.3125 提高到 0.5000。

一些公共 benchmark 也有提升。4B 的 HumanEvalFix JavaScript 增加 10.98 percentage points，CodeXGLUE defect detection 增加 7.03 points。9B 的 BBH Word Sorting 从 0.240 提高到 0.576。72B 的 APPS Introductory 增加 6.25 points，LiveCodeBench Execution 增加 5.47 points。

作者也报告了反例。9B 在 HumanEvalFix Python 的独立确认中下降 8.33 points，confidence interval 跨过零。现有证据支持 selected-task transfer，尚不能推导出所有能力都会同步提高。

## Long-horizon scientific RL

论文进一步在 LAPS 与 MITgcm-biogeo 上做 online RL。每个 repair episode 可能跨越几十轮、数万 tokens，最终 reward 来自编译后的真实 scientific simulation。一个 episode 可能重建 Fortran solver 并运行完整模拟，执行时间差异很大。

系统让 rollout 与 optimization 使用不同 GPU，generation 可以在 optimizer 工作时继续进行。Agent-loop workers 是轻量 CPU processes，大量 episodes 并发运行。由于 rollout 使用的 policy 可能落后一个 update，训练通过 token-level truncated importance sampling 修正 staleness。

每个 prompt 生成八条 trajectories，使用 group-relative advantage，但省略标准差归一化，避免 near-bimodal rewards 放大孤立成功样本。PPO-style ratio 采用 asymmetric clipping，给低概率但有效的 repair action 更多上升空间。

这个实验的价值在于它暴露了 agent RL 的真实系统形态：reward 极晚、episode 极长、execution cost 不均衡，generation、simulation 与 optimization 使用不同资源。Budget cutoff、scientific failure 与 infrastructure failure 必须分开建模，否则训练信号会被污染。

## 边界

当前任务供给严重偏向 repair 与 implementation，discovery、reproduction、calibration 和 acceleration 的覆盖仍然有限。Environment construction 也需要 domain expert 审核 module boundaries、observables 和 tolerances，规模化速度会受专家时间约束。

SFT 的公共 benchmark 结果属于选择性增益，部分任务会退化。ScienceIDE-Hard 也主要覆盖物理模拟类 codebases，距离生命科学实验、复杂数据分析与开放式发现还有很长距离。

即便如此，ScienceIDE 给出了一套很完整的 agent post-training data model：repository、runtime、scientific contract、task factory、trajectory、verifier reward 与 resource telemetry 都有明确位置。对于希望把真实软件与领域工作变成长期训练供给的团队，这篇论文值得仔细看。

## 另外两篇

第二篇是 **Rethinking Critic Learning in PPO: Understanding and Mitigating Value Flattening**。

作者发现，真实 Monte Carlo return 会在 reasoning trace 的中间状态明显变化，critic prediction 却趋于平坦。原因包括 critic loss 隐含的 variance penalty，以及相邻 token states 产生大量高度相关的重复梯度。SP3O 只在每条 response 的少数、彼此分离的 states 上训练 value loss，三个 state targets 就能缓解 Value Flattening，并改善 Qwen3-Base 的 policy learning。

第三篇是 **ProgramDistill: From Interactive Web Apps to Verifiable Reference-Guided SWE Tasks**。

它让 coding agent 观察一个可运行的 reference web app，再把发现的功能重建到不完整应用中。Mine-craft-patch pipeline 在 26 个 apps 中发现 1,975 个 replay-verified behaviors，并自动构造 4,063 个 tasks。任务深度增加时成功率明显下降，为浏览器探索、隐式 specification recovery 与 long-horizon coding 提供了可控 benchmark。

论文：<https://arxiv.org/abs/2609.19134>

另外两篇：<https://arxiv.org/abs/2609.18708>、<https://arxiv.org/abs/2609.18805>
