---
title: "Paper Digest: 2026-07-07"
categories: [Paper Digest]
tags: [AI, Coding Agents, Interpretability, Post-training, Software Engineering, Code LLMs]
---

今天最值得看的 paper，我会选 **Latent Programming Horizons in Coding Agents**。

这篇 paper 问了一个很有意思的问题：当 coding agent 一步步读代码、改文件、跑测试时，底层 language model 的 hidden state 里，到底有没有形成对程序状态的内部判断？

作者没有只看最终 patch 是否通过测试。他们把 agent 运行过程拆开，直接在模型 residual stream 上训练 linear probes，去预测一些很具体的程序属性：

- 当前代码能不能 parse
- 当前代码能不能通过 test suite
- 当前 edit 是否减少 failing tests
- 当前 edit 是否引入 regression

结果挺锋利。

这些 probe 在两个模型和两个 benchmark 上都能读出有效信号。对 correctness 的预测，AUC 最高到 **0.83**。

更有意思的是，这些内部表示会提前跑到 agent 行为前面。

也就是说，在某些运行中，模型 hidden state 对未来 edit 的结果已经有可预测信号，即使这一步 edit 还没有真正写到磁盘上。作者把这叫作 **latent programming horizon**。他们报告这个提前量可以延伸到大约 **25 steps**。

这对 coding agent 很重要。

今天大部分 agent eval 仍然站在外面看结果：patch 过没过，test 绿没绿，SWE-bench resolve 没 resolve。这个视角当然重要，但它太晚了。等 patch 已经写完、测试已经跑完、上下文已经烧掉，很多失败路径其实早就发生了。

如果 agent 的内部状态已经携带了“这条路可能走不通”的信号，harness 就有机会更早介入：

- 提前停止低质量 repair loop
- 在 agent 写坏代码前切换策略
- 用 probe 当轻量 evaluator
- 从完整 agent trace 里提取更密的训练信号
- 给 post-training 提供比 final pass/fail 更细的 reward 或 filtering signal

这篇 paper 最吸引我的地方，是它把 coding-agent observability 往模型内部推进了一步。

过去我们常说 agent 有 planning、search、debugging、reflection。但这些词很多时候停在外部行为层：它写了 plan，它运行了 tests，它改了 patch。Latent Programming Horizons 提供了一个更硬的测量方式：模型在每一步内部是否已经线性编码了程序状态和未来结果。

当然，这还不是一个完整产品方案。

Probe 能读出信号，不代表 harness 马上能稳定利用这个信号。AUC 0.83 也不是万能 oracle。不同模型、不同 scaffold、不同任务类型下，latent horizon 的长度和可转移性还需要更多验证。

但方向很值得追。

如果 coding agent 未来要做得更可靠，光看 final answer 不够。我们需要看见它在运行中如何形成判断、何时开始偏航、哪些中间表示能预测后续失败。

今天第二篇是 **UI-MOPD: Multi-Platform On-Policy Distillation for Continual GUI Agent Learning**。

这篇来自 HuggingFace Daily Papers，关注 GUI agents 的 continual learning。

GUI agent 的难点在于，不同平台有不同交互习惯。桌面、移动端、网页、应用之间，action pattern 不一样，数据覆盖也不一样。一个模型如果直接做 joint training 或 continual training，很容易混淆平台行为，或者在学新平台时忘掉旧平台能力。

UI-MOPD 的做法是构建 Uni-GUI 数据集，并使用 multi-teacher on-policy distillation。系统根据当前环境选择 platform-specific teacher，再把对应平台的行为先验蒸馏到 shared policy 中。

论文在 OSWorld 和 MobileWorld 上报告 success rate 分别为 **38.2%** 和 **12.0%**。

我觉得它的关键不在分数本身，而在训练结构：多个 teacher、on-policy trajectories、platform-conditioned distillation、continual learning。GUI agent 的 post-training 很可能会越来越像这种问题，每个环境都有自己的行为语法，模型需要学会共享能力，同时保留局部习惯。

第三篇是 **Teaching Code LLMs to Reason with Intermediate Formal Specifications**。

它提出 SpecCoder，训练 Code LLM 在程序内部生成 executable checkpoint specifications。

普通自然语言解释很容易说得通顺，但不一定能抓住程序语义。Whole-program precondition 或 postcondition 又太粗。SpecCoder 选择中间层：在关键 program points 插入可执行 assertions，让模型把程序理解变成可检查的中间证据。

训练信号来自 validated reference programs、behavior-changing mutants，以及多轮 specification refinement traces。论文在 HumanExec benchmark 上报告，SpecCoder 对 Qwen2.5-Coder 系列模型带来明显提升：

- inline-specification correctness 最高提升 **55.8%**
- completeness 最高提升 **358.1%**
- executable assertion validity 最高提升 **26.6%**

这和第一篇放在一起很有意思。

Latent Programming Horizons 研究 hidden state 里是否已经有程序状态信号。SpecCoder 研究如何把程序理解外化成 executable checkpoints。一个偏内部观测，一个偏外部证据。两者都指向同一个问题：coding agent 不能只靠最后一次测试结果来理解自己是否走对了。

今天这三篇放在一起，主题很集中：

**coding agent 的下一阶段，会越来越依赖运行过程里的可观测信号。**

内部 hidden state、on-policy distillation trace、executable intermediate specs，都在把“agent 到底知道什么、什么时候知道、能不能把这种知道用起来”变成可研究的对象。

如果你关心 coding agents、SFT/RL、post-training、agent observability，今天先读 `2607.05188`。

## 今日 3 篇精选

### 1) Latent Programming Horizons in Coding Agents
- 链接: https://arxiv.org/abs/2607.05188
- 摘要速读: 在 coding-agent 运行过程中用 linear probes 读取模型 residual stream，预测代码 parse、test、regression 和未来 edit outcome。
- 为什么重要: 它说明 agent hidden state 可能提前携带未来 patch 成败信号，为 agent steering、早停、reward modeling 和 trace-based post-training 提供了新入口。

### 2) UI-MOPD: Multi-Platform On-Policy Distillation for Continual GUI Agent Learning
- 链接: https://arxiv.org/abs/2607.04425
- 摘要速读: 用 multi-teacher on-policy distillation 训练跨平台 GUI agent，通过 platform-conditioned distillation 保留不同环境的行为先验。
- 为什么重要: GUI agent 的 post-training 需要处理平台差异、行为混淆和 continual learning forgetting，这篇给了一个很直接的训练结构。

### 3) Teaching Code LLMs to Reason with Intermediate Formal Specifications
- 链接: https://arxiv.org/abs/2607.04232
- 摘要速读: 训练 Code LLM 在程序内部生成 executable checkpoint specifications，用中间 assertions 支撑 correctness checking 和 repair。
- 为什么重要: 它把程序理解变成可执行证据，为 code-agent verifier 和 dense supervision 提供更细的信号。

## 一句话结论

今天最强的新信号是：**coding agent 的观察窗口需要进入运行过程内部，不能只停在最后的 patch 成败。** `2607.05188` 值得先读。
