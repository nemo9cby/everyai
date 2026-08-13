---
title: "Paper Digest: 2026-08-13"
categories: [Paper Digest]
tags: [AI, Code Agents, Software Engineering, Tool Architecture, RLVR, Triton]
---

今天最值得看的 paper，我会选 **The Devil Is in the Interface: Evaluating How Tool Architecture Shapes Coding Agent Behavior**。

Coding agent 的工具越来越多。Shell、文件读写、代码搜索、patch、Python execution、scratchpad、subagent，几乎每个 harness 都有自己的组合。讨论这些系统时，我们通常先问模型能调用哪些工具，或者 benchmark resolve rate 提高了多少。

这篇论文问了一个更细、也更接近系统设计的问题：当可用信息和操作能力大致相同时，工具以什么接口暴露给模型，会不会改变 agent 的行为？

作者把这个维度称为 **tool architecture**。他们设计了六种 capability 相近的接口，在三个 actor models 上跑了 11,700 条 repository-level coding trajectories。结果很清楚：总 resolve rate 大体接近，稳定性、搜索范围、执行步数和 token cost 却出现显著差异。

这意味着 tool API 本身就是 agent policy 的一部分。

## Tool capability 和 tool architecture

论文借用了 software engineering 里的区分。

Tool capability 描述 agent 能获得哪些信息、执行哪些动作。Tool architecture 描述这些能力如何被组织，以及模型通过怎样的接口访问它们。

例如，一个只有 Bash 的 agent 可以使用 `grep` 搜索 repository、用脚本编辑文件、运行 tests，也可以临时写 Python 完成复杂操作。另一个 agent 拥有独立的 search、view 和 edit tools。两者能够完成的底层动作高度重叠，交互方式却完全不同。

Bash 要求模型自己组合命令，处理 quoting、输出范围和失败恢复。Atomic tools 把高频操作包装成有明确参数与边界的 primitives。Python CodeAct 让模型用一段可执行代码完成多步操作。NLSearch 允许模型用自然语言表达检索意图，再由 search subagent 调用 Bash 查找相关代码。

同一组能力经过不同接口，会给模型形成不同的 action space、反馈粒度与错误表面。

## 六种工具架构

论文比较了六个 setups：

- **BashOnly**：所有 navigation、search、editing 和 execution 都通过通用 shell 完成
- **Atomic**：在 Bash 之外提供 search、bounded file view、targeted replacement 和 file creation 等结构化工具
- **NLSearch**：增加自然语言 repository search，由同一个 actor model 作为 subagent 使用 Bash 查找 snippets
- **Python**：模型输出可执行 Python blocks，完成遍历、读取、编辑与验证
- **HypoTrack**：在 Bash 上增加记录与更新 hypothesis 的文本工具
- **Scratchpad**：在 Bash 上增加保存 intermediate reasoning 的文本工具

作者刻意限制 capability difference。NLSearch 没有使用 embedding index，它的底层仍然是 `grep` 等 Bash operations。对 100 个 Python actions 的抽样检查显示，97% 对应 BashOnly 已经具备的操作，或者 Bash agent 本来就可以创建并执行的 Python script。

HypoTrack 和 Scratchpad 也没有增加 retrieval、memory management 或新的 task information。它们只给模型一个显式记录中间状态的地方。

这种控制设计很重要。否则某个 setup 变好时，很难判断增益来自新信息、新动作，还是接口组织本身。

## 11,700 条 trajectories 怎样产生

主实验使用 SWE-bench Live。作者从 100 个 repositories 中随机抽取 25 个，每个 repository 最多保留五个 issues，最终得到 65 个 problem instances。

三个 actors 分别是 Qwen3Coder-30B、Kimi K2.5 和 Claude Sonnet 4.5。每个 actor 与 tool setup 的组合，在每个 issue 上执行 10 次独立 rollouts。重复尝试让实验可以测量 aggregate resolve rate，也可以观察同一任务多次运行时的 consistency、exploration diversity 和 solution diversity。

作者还在 SWE-bench Verified、SWE-bench Pro 与带完整 stack traces 的 debugging split 上做了补充实验，用来检查主要现象能否扩展到 issue resolution、feature implementation 和 debugging。

## Resolve rate 接近，运行方式差很多

第一个结果看起来平淡，却决定了整篇论文的解释空间：对同一个 actor，六种架构的总体 task resolve rate 大体相似。

这是实验希望看到的控制结果。工具能力接近时，接口没有凭空创造大量新能力。接下来观察到的差异主要落在 non-functional properties 上。

对实际 agent service 来说，这些性质很硬。相同成功率下，一个系统可能更稳定，一个更善于探索，一个花费更少。只看 pass rate 会把这些工程差异全部压扁。

## Atomic tools 提高重复尝试的稳定性

Atomic 是唯一在三个 actor 上都相对 BashOnly 改善 pass^5、pass^7 和 pass^9 的 setup。提升幅度对较弱模型更明显。

在 Qwen3Coder-30B 上，Atomic 根据不同 k 值带来 0.059 到 0.074 的绝对提升。Kimi K2.5 和 Sonnet 4.5 的增益较小，但方向保持一致。论文把最高相对改善概括为 4.7 倍。

一个合理解释是，结构化 primitives 减少了低层执行错误。较弱 actor 更容易在 command composition、file range、editing syntax 或 output handling 上积累小失误。Atomic interface 把常见操作收敛到参数明确的路径，让重复 rollouts 更少被偶然的 shell mistake 带偏。

这类收益很容易被单次 benchmark run 漏掉。Average resolve rate 可能变化有限，多次尝试的可靠性已经改善。

## Natural-language search 扩大探索，也带来噪声

NLSearch 允许 actor 用自然语言描述想找的代码，再由 search subagent 使用 Bash 搜索 repository。它没有引入 semantic embeddings，依然能让 agent 访问的文件数量增加超过 11%。

接口提高了表达层级。Actor 可以描述概念、行为或可能的 implementation location，无需先猜准确 identifier，再手工组织一串 `grep` commands。

代价也很直接。搜索范围变宽后，返回内容包含更多噪声。访问更多文件并不自动等于更准确的 localization，探索 breadth 与 context precision 之间仍需要平衡。

这个结果对 repository-scale agents 很实用。Search tool 的价值很大一部分可能来自 query interface 和 delegation structure，semantic index 只是其中一个组成部分。评估 retrieval improvement 时，最好把接口变化与新增知识源分开测试。

## Python CodeAct 大幅降低交互成本

Python setup 达到相近 task performance，同时平均减少 41.6% 的 steps 和 56.3% 的 token usage。

原因并不神秘。一次 Python block 可以遍历多个文件、聚合结果、进行条件过滤、修改内容并输出紧凑摘要。Tool-call interface 往往需要多轮 request、result、follow-up 才能完成同样的操作。每一轮还会把 tool schema、调用参数和返回结果带进 context。

Code execution 把一部分 orchestration 压缩进模型生成的程序里。Agent 依然要决定做什么，环境交互的 round trips 明显减少。

这对大规模 rollout collection 很重要。训练数据成本不仅来自最终 answer tokens，也来自长 trajectory 中反复出现的 observations 和 tool protocol。若某种 interface 可以保留成功率并减少一半以上 token usage，它会直接改变 RL 或 evaluation pipeline 的可承受规模。

## Scratchpad 没有自动改变 reasoning

HypoTrack 与 Scratchpad 的影响有限。给模型一个额外文本区域，并没有稳定改变 actors 的底层 reasoning behavior。

论文的实现有意保持轻量，它们没有强制 reasoning policy，也没有 retrieval 和 memory management。这个结果适合谨慎解读：一个可写入的文本工具本身很弱。要让 cognitive scaffold 产生效果，可能需要明确的更新规则、触发条件、verification loop，或者让后续决策真实依赖其中的状态。

对于长期 agent memory，这也是一个提醒。存储入口很容易做，决定何时写、写什么、怎样检索、何时删除，才会真正塑造行为。

## 对 code-agent post-training 的启发

Tool architecture 会进入 trajectory distribution。

在 BashOnly 下收集的 SFT data，包含大量 command composition、shell recovery 与细粒度 observations。Atomic setup 会减少这类 execution noise。NLSearch 会产生更宽的 repository exploration。Python CodeAct 会把多个动作压缩进代码 block，并显著缩短 interaction horizon。

这些差异随后会被训练进 policy。模型可能学会某个 harness 的动作语法、搜索习惯和错误恢复方式，并在换接口后失去一部分表现。

下一组值得做的实验是完整的 cross-interface matrix：

- 在 Bash trajectories 上训练，分别到 Atomic、NLSearch 和 Python 中测试
- 在 Python CodeAct 上训练，检查模型能否回到细粒度 tool calls
- 固定 solved tasks，比较不同架构产生的 trajectory entropy、zero-reward ratio 和 credit-assignment length
- 把 interface randomization 加入 rollout collection，观察它能否提升 scaffold transfer
- 分别训练 problem-solving policy 与 interface adapter，测量 capability isolation

这会把 harness engineering 与 post-training data design 接起来。Tool choice 同时决定线上系统的行为，也决定模型看过怎样的训练世界。

## 应该怎样解读

主实验只有 65 个 SWE-bench Live issues，尽管每个 setting 做了大量重复 rollouts，任务覆盖面仍然有限。Actors 也只有三个，模型对 tool schema 的训练历史可能影响结果。

Capability equivalence 只能近似成立。Python、Bash 与 NLSearch 的表达能力在理论上高度重叠，模型实际使用这些能力的难度并不相同。这正是 tool architecture 的一部分，也让严格分解机制变得困难。

论文更适合支持一个系统结论：agent evaluation 需要把 interface 当作一等变量。它没有给出适用于所有 agents 的唯一最佳工具组合。

## 为什么今天选它

这篇论文把一个经常被当作实现细节的问题，做成了大规模 controlled study。

六种 tool architectures、三个 actors、11,700 条 trajectories，给出了三个很有操作性的结果：Atomic primitives 提高重复运行稳定性，natural-language search 扩大 repository exploration，Python CodeAct 显著减少 steps 与 tokens。

模型、prompt 和 benchmark 相同，接口仍会塑造行为。对于构建 coding agents、收集 post-training trajectories 或比较不同 harness 的团队，这个变量已经不能藏在实验配置附录里。

## 另外两篇

第二篇是 **Parameter Exploration for RLVR via Variational Learning**。

作者提出 Perturbed Parameter Policy Optimization，简称 3PO。常见 RLVR exploration 主要调整 action-space sampling，例如 temperature。3PO 从 learned parameter posterior 中采样不同 rollout policies，让探索发生在 parameter space。OLMo-3-1025-7B 与 Qwen2.5-Math-7B 的数学和代码实验中，它在近似相同 FLOPs 下持续优于 GRPO，并减少 zero-advantage groups 与 malformed rollouts。对 distributed RL 系统来说，policy variants 如何分配到 rollout workers，以及怎样控制 staleness 和通信开销，会是很实际的下一步。

第三篇是 **RealisticTritonBench: A Benchmark for Triton-Kernel Generation in Real-World AI Frameworks**。

它从流行 AI frameworks 中真实修改 Triton kernels 的 pull requests 构建任务。生成结果会被集成回原 framework，通过 end-to-end tests 验证。此前许多 benchmark 集中在 PyTorch-to-Triton translation、isolated kernel speed，或者依赖容易被模型绕过的手写 checks。RealisticTritonBench 把问题拉回 production context：requirement、integration、correctness 与最终 framework performance 要一起成立。领先 LLMs 在这些任务上仍然困难。

论文：<https://arxiv.org/abs/2608.11386>

另外两篇：<https://arxiv.org/abs/2608.09805>、<https://arxiv.org/abs/2608.12004>
