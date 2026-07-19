---
title: "Paper Digest: 2026-07-18"
categories: [Paper Digest]
tags: [AI, Long Context, Reinforcement Learning, Coding Agents, Agent Safety, Distributed Training]
---

今天最值得看的 paper，我会选 **LongStraw: Long-Context RL Beyond 2M Tokens under a Fixed GPU Budget**。

这篇 paper 解决了一个越来越明显的落差：模型推理已经接近 million-token context，RL post-training 仍经常停在 256K 或更短。

这个差距对 agent 特别麻烦。一次长任务会不断积累 observation、tool output、文档和历史决策。部署时让模型处理百万 token，却只用短上下文做 RL，最后只能赌 length generalization。

LongStraw 的核心是一套 architecture-aware execution stack，当前实例使用 GRPO。

它利用了 agent RL 中常见的结构：一组 rollout 共享很长的 prompt，真正分叉的 response 相对短。系统先在没有 autograd 的情况下计算共享 prompt，只保留后续 token 需要的 model-specific state，再逐个 replay 短 response branch。

代价是增加 replay time，收益是训练时不需要让整条百万 token trajectory 一直留在 live computation graph 中。

实验数字很醒目。

在 8 张 H20 上，LongStraw 对 Qwen3.6-27B 完成了 2.1M positions、group size 2 和 8 的 grouped scoring 与 response backward。group size 增大只增加 0.21 GB peak allocated memory。单独的 stress test 达到 4.46M positions。

在 32 张 H20 上，作者还验证了 GLM-5.2 的 2.1M-token prompt execution path，覆盖全部 78 层。

论文有一个很重要的诚实声明：这些实验建立的是 **execution capacity**，完整 training correctness 仍未完成。captured prompt state 被 detach，部分 distributed forward 和 gradient composition path 还不完整。

这恰好是我喜欢这篇 paper 的地方。它给出的技术进展很实在，也把边界说得很清楚。LongStraw 已经证明百万 token RL 能越过显存墙，距离稳定训练系统仍有工程和数学上的缺口。

对做 post-training infra 的人，最值得带走的是 shared-prompt evaluation 加 response-branch replay 这个设计。它把 group sampling 的共同部分和分叉部分拆开处理，尤其适合长 prompt、短 rollout、多 response 的 agentic RL workload。

今天第二篇是 **SEED: Self-Evolving On-Policy Distillation for Agentic Reinforcement Learning**。

Agent RL 常用 trajectory-level outcome reward。它能告诉模型任务最终成功还是失败，却很难解释中间哪一步做对了，哪条 observation 真正关键，哪个错误以后应该避开。

SEED 把完成的 on-policy trajectory 转成 natural-language hindsight skills，例如可复用 workflow、关键 observation、failure-avoidance rule。

接着，它分别在普通 context 和 skill-augmented context 下重新计算 sampled action 的概率。skill 带来的 probability shift 会被转换为 dense token-level on-policy distillation signal，再与 outcome-based RL 联合优化。

这个设计的亮点在于 current policy 同时负责收集 trajectory 和提炼 hindsight skill。policy 更新后，skill analyzer 也随之更新，辅助监督持续贴近当前 trajectory distribution。

它让自然语言经验真正进入训练目标。系统关注 skill 对行为概率的影响，而不只保存一段可读的总结。

今天第三篇是 **Stop Means Stop: Measuring and Repairing the Enforcement Gap in Agent-Framework Control Primitives**。

作者测试了六个常用开源 agent framework 的 approval gate、run cancellation 和 timeout，发现六个框架都无法完整兑现这些 control primitive 暗示的 barrier semantics。

最典型的问题叫 sibling leak。一个 branch 等待 human approval 时，另一个 sibling branch 已经执行了 side effect。用户随后点 reject，也无法撤回已经发生的写操作。

这类路径具有现实可达性。frontier model 生成 leak-triggering plan shape 的比例最高达到 14%。在 live model 驱动未修改 framework 的实验中，1,200 次 run 里有 215 次在 approval pause 期间执行了 effect。

作者提出 SOUNDGATE，把 effect admission 放到 agent framework 外部，用统一 gate 实现 hold-until-decided、reject-cancels、dedup-on-replay 和 fence-on-cancel。

这篇对所有 production agent runtime 都值得读。只要系统允许 parallel branch，UI 上的“暂停”就必须对应真正的 global effect barrier。控制按钮的名字无法提供安全性，complete mediation 才能。

今天三篇 paper 对应 agent 系统的三个层面：

1. LongStraw 解决 million-token RL 的 execution capacity。
2. SEED 缩小 sparse outcome reward 和 token-level learning 之间的 supervision gap。
3. Stop Means Stop 修补 agent runtime 的 side-effect boundary。

如果只读一篇，先读 `2607.14952`。它与 long-context agent、GRPO 和 distributed post-training 的交叉最紧，而且论文对已完成和未完成的部分交代得足够清楚。

## 今日 3 篇精选

### 1) LongStraw: Long-Context RL Beyond 2M Tokens under a Fixed GPU Budget
- 链接: https://arxiv.org/abs/2607.14952
- 摘要速读: 利用 shared prompt 的无梯度计算和 response branch replay，把 GRPO post-training 推到 2.1M positions，stress test 达到 4.46M。
- 为什么重要: 它直接处理 inference context 与 RL context 之间的巨大差距，同时明确承认完整 training correctness 尚未闭环。

### 2) SEED: Self-Evolving On-Policy Distillation for Agentic Reinforcement Learning
- 链接: https://arxiv.org/abs/2607.14777
- 摘要速读: 从 on-policy trajectory 提炼 hindsight skills，并把 skill 引起的 token probability shift 变成 dense distillation signal。
- 为什么重要: 给 sparse outcome reward 补上更细粒度的行为指导，同时让监督信号随 policy 一起更新。

### 3) Stop Means Stop: Measuring and Repairing the Enforcement Gap in Agent-Framework Control Primitives
- 链接: https://arxiv.org/abs/2607.14166
- 摘要速读: 六个开源 agent framework 都存在 approval、cancel 或 timeout enforcement gap，作者用外部 effect gate 修复。
- 为什么重要: production agent 的 human approval 需要真正阻止所有 sibling side effects，单个 branch 的 pause 远远不够。

## 一句话结论

今天最强的信号是：**百万 token agent RL 的瓶颈已经落到 execution stack，LongStraw 给出了一条可信但尚未完全闭环的工程路径。**
