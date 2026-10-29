---
title: "Paper Digest: 2026-10-29"
categories: [Paper Digest]
tags: [AI, LLM, Tool Use, SFT, Code Agents]
---

今天最值得看的 paper 是 **Small agents should know what their tools do**。

小模型已经能调用工具，却经常缺少对工具副作用的理解：重复写入、删除前不确认、反复读取已经掌握的状态，或者把几次存在依赖关系的调用拆成多个回合。训练数据通常只记录工具名、参数和返回值，工具对外部世界造成的影响仍然藏在自然语言里。

这篇论文提出 EffectWorld，用一组可执行的类型声明描述每个工具的行为。每个工具都标明它读取和写入哪些资源、操作是否幂等，以及调用会产生什么 effect。Runtime 因此可以机械验证一条 trajectory 是否安全，也可以把多个相互依赖的调用编译成一次执行的短程序。

## Effect type 给 tool use 增加可验证语义

EffectWorld 的工具声明支持三项能力：

1. Replay verifier 可以根据读写集合和幂等性检查 trace，无需人工标签或 judge model。
2. Executor 可以在一个回合中运行包含多次依赖调用的 tool program，减少来回推理。
3. Compiler-style optimizer 可以识别重复读取、冗余写入和危险顺序，再把优化后的轨迹转成经过验证的训练样本。

这让 agent training 获得了比文本模仿更强的监督。模型学习的目标包含任务完成、effect safety 和调用效率，监督信号可以由执行语义直接产生。

## Qwen3-4B 从 53% 提升到 99%

作者用 EffectWorld 生成的数据训练 LoRA adapter。在 Qwen3-4B 上：

- 分布内 task success 从 **53% 提升到 99%**。
- 面对未见过的任务类型和 distractor tools，从 **52% 提升到 81%**。
- 在新领域中从 **74% 提升到 83%**。
- 输出 token 约减少到原来的三分之一。

收益也迁移到了公开 benchmark。Berkeley Function-Calling Leaderboard 上，adapter 保持了 base model 的 live accuracy，并把 multi-turn accuracy 从 21.9 提升到 26.6。加入 near-miss decline 样本后，整体平均分超过 base model，同时维持 irrelevance detection。

## 公开 tool-use 数据可能在教模型拒绝调用

论文中最值得复查的发现来自数据审计。许多公开 tool-use 数据包含 refusal templates，这些模板会让小模型学会在本应调用工具时选择拒绝。加入这类数据后，BFCL non-live accuracy 从 88.0 跌到 31.4；过滤后恢复到 84.8。

作者进一步比较了训练目标。针对 verified negatives 的 suppression 和 preference objectives 没有超过 SFT-only control。真正推动行为变化的是高质量、针对性的 positive examples，而且模型改进的行为与示例展示的行为高度对应。

这对 post-training 有一个直接提醒：复杂 objective 无法补偿错误的数据语义。先明确工具会改变什么状态，再构造能够展示正确决策和正确调用顺序的正样本，往往更有效。

## 对 code agent runtime 的启发

EffectWorld 的思路可以直接进入真实 agent 系统：

- Tool schema 增加 read set、write set、idempotence、reversibility 和 mutation scope。
- Approval policy 根据 effect 和资源范围分配确认、sandbox 或直接执行。
- Trace optimizer 合并安全的依赖调用，删除无意义的重复读取。
- 数据流水线用 replay verifier 筛除错误 trajectory，再生成修正后的 positive demonstrations。
- Agent RL 可以把 effect violation、冗余调用和不可逆副作用纳入 reward 或 verifier。

对 OpenClaw 这类同时拥有 filesystem、message、browser 和 cron 工具的 runtime，effect metadata 还能连接权限控制、失败恢复与训练数据采集。工具声明一旦包含可执行语义，安全边界和数据质量就拥有共同的基础设施。

## 今天另外两篇相关论文

**LongZero: Enhancing Long-Context Reasoning in LLMs from Zero Data** 让模型先生成紧凑的 question、answer 和 supporting evidence，再把 evidence 扩展成长篇多文档 context。它结合自生成 multi-hop reasoning、multi-round retrieval 和 GRPO，在不依赖外部语料或强教师的条件下提升 long-context reasoning。

**TAO: Tool-Aware Joint Scheduling for LLM-Based Agentic Serving** 把 agent serving 建模为 GPU inference 与分布式 CPU tools 的联合调度问题。它同时考虑 tool-return readiness、带时间衰减的 cache credit 和 token footprint，最高降低 26.8% latency、36.3% TTFT，并提高 29.5% decode throughput。

论文：<https://arxiv.org/abs/2610.35981>

另外两篇：<https://arxiv.org/abs/2610.30132>、<https://arxiv.org/abs/2610.37230>
