---
title: "Paper Digest: 2026-08-24"
categories: [Paper Digest]
tags: [AI, Code Agents, Formal Verification, Lean, RISC-V, Agent Systems]
---

今天最值得看的 paper，我会选 **AI with Authority, from Application to Silicon**。

论文报告了一个相当激进的工程实验。一个研究者使用消费级 AI subscriptions，指挥一支小型 agent fleet，在五周内完成应用代码、verified compiler、executive 和 RISC-V processor，并把芯片送入 community silicon shuttle tapeout。过程中没有人类编写 RTL，也没有人工逐条审阅 proof。

真正让这个结果站得住的，是贯穿整个工程栈的 machine verification。Agent 可以高速生成代码、定理与证明，但每个关键 claim 都要交给无法被语言技巧说服的 proof kernel。论文把这套工作纪律称为 Salt method。

## Agent 需要一个不可讨价还价的裁判

普通 code agent workflow 主要依赖 tests、review 和 retry。它们很有效，也有清晰的边界。测试覆盖只能检查已表达出来的行为，review 的吞吐又远低于 agent 生成速度。系统规模扩大后，人类很容易成为验证瓶颈。

Salt method 让数学 claim 以 kernel-checked artifact 的形式在 agents 之间流转。Lean 4 kernel 检查 formal proofs，silicon boundary 则使用 SAT-checked equivalence。生成证明的 agent 无法靠一段听起来合理的解释越过这道门。Kernel 接受或拒绝，结果明确且可重复。

这改变了人类注意力的分配方式。研究者主要负责 specification、design 和 ruling，把可机械检查的正确性判断交给 verification infrastructure。Agent 的生成速度越快，这种分工越重要，因为每一次自动拒绝都节省了一次潜在的人类审查。

## 验证工件构成一张 authority graph

这项工作的核心可以理解成一张 verified artifact graph。应用、compiler pass、executive、RTL 与物理实现彼此连接，每条关键边都有对应的 proof obligation 或 equivalence check。

一项上游转换如果发生错误，下游结果即使能运行，也无法获得完整 authority。只有当链路上的检查全部通过，最终 silicon artifact 才继承整条路径的可信度。

这种结构很适合 multi-agent system。不同 agents 可以负责实现、证明、反例搜索和修复，协作接口是一组可验证工件。Agent 之间无需彼此相信，也无需共享完全相同的上下文。它们只需要遵守同一套 machine-checkable contract。

## 错误账本比完美演示更有价值

论文公开了 theorem provenance、预注册 token meter、人类投入时间的下界，以及 append-only error ledger。Ledger 的 catch 编号至少达到 #256，最终 accepted record 中没有错误 proof。

这个数字很重要。它说明 agent fleet 并没有稳定地产生完美结果。系统的可靠性来自高频失败能够被检查器捕获、留下记录、触发修复，并且无法悄悄进入正式工件。

对于 agent engineering，这比单个成功率更有信息量。一个生产系统应该记录：

- 哪个 agent 提出了 claim
- 使用了哪些输入和 tool versions
- 哪个 verifier 接受或拒绝了它
- 失败原因如何影响下一次尝试
- 最终 artifact 继承了哪些验证链路

这些记录既是 audit trail，也可以成为 post-training data。被 kernel 拒绝的 proof、SAT counterexample 和 design revision 都是高质量 process supervision，能够训练模型更早识别 proof obligation、发现 specification 缺口，并减少重复犯错。

## 对 code agent harness 的启发

这篇论文最值得借鉴的部分，是它把 verification 放进 agent control loop 的中心。Verifier 的角色覆盖 reward、权限边界和协作协议。

一次 agent action 只有产生了可检查工件，才能推动全局状态前进。模糊的自然语言结论不会自动升级成已验证事实。后续 agent 获取上下文时，也可以直接读取 artifact 和 provenance，降低错误摘要沿着协作链传播的风险。

这套模式可以扩展到 compiler 和 hardware 之外：

- repository agent 可以提交 patch、tests、static-analysis certificate 和 dependency provenance
- data pipeline agent 可以提交 schema checks、lineage 和 reproducible query results
- RL environment generator 可以提交 executable invariants 与 verifier coverage
- distributed training agent 可以提交配置约束、checkpoint hashes 和数值一致性检查

可验证边界越接近关键状态变化，agent autonomy 就越容易安全扩大。

## 为什么今天选它

很多 agent paper 展示了更高 benchmark 分数，这篇展示了一套完整的 authority architecture。一个人能够指挥 agents 完成横跨 software、formal methods 和 silicon 的真实项目，成果还进入了物理制造流程。

更重要的是，论文把成功背后的失败、token、人类时间和 verification provenance 一起公开。它给 code agents 提供了一个清晰方向：生成能力负责扩张搜索空间，proof kernel 和 executable verifier负责守住事实边界，人类继续掌握目标、设计和最终裁决。

## 另外两篇

第二篇是 **AgentMercury: Your Agent Can Synthesize Verifiable Environments for Business Scenarios at Scale**。

AgentMercury 根据高层 business scenario 构造 persistent executable world，其中包含 entities、services、tools、state 和跨服务 invariants。作者生成了 4,783 个环境，覆盖 14 个行业和 50 个国家，并将它们用于 RL。Qwen3.5-4B 在 EnterpriseOps-GYM 上由 12.3 提升至 15.7，在 AIME26 上由 45.9 提升至 56.0。环境构造能力本身也可以学习，Qwen3.5-35B-A3B 的 held-out executable-world authoring success 由 3.3% 提升至 83.3%。这给 scalable agent RL 提供了一种 world-first 的 synthetic environment recipe。

第三篇是 **Beyond Fault Localization: A Trajectory-Level Study of LLM Agents for Microservice Root Cause Analysis**。

论文分析了 3,500 条 microservice diagnosis trajectories，发现 final answer 正确仍可能伴随薄弱的证据链。成功调查通常停留在 fault-impact surface 上，持续使用 retrieved telemetry，并随着搜索深入扩展 query repertoire。作者据此设计 DiagGuard，在定位之前做 observation grounding，在结论之后做 evidence verification，使独立设置中的 Acc@1 由 43.5% 提升至 52.5%。这是一份很适合 process reward 和 SRE agent evaluation 的 trajectory-level taxonomy。

论文：<https://arxiv.org/abs/2608.21356>

另外两篇：<https://arxiv.org/abs/2608.20634>、<https://arxiv.org/abs/2608.21310>
