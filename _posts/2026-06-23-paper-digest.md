---
title: "Paper Digest: 2026-06-23"
categories: [Paper Digest]
tags: [AI, Terminal Agents, Coding Agents, RL, SFT, Post Training]
---

今天最值得看的 paper，我会选 **Tmax: A simple recipe for terminal agents**。

这篇论文处理的是一个很现实的问题：terminal agent 已经变成 LLM 最重要的下游形态之一，但公开的训练 recipe 还不够清楚。

我们现在说 coding agent，很多时候说的其实就是 terminal agent：

- 它要读文件
- 它要跑命令
- 它要改代码
- 它要看测试输出
- 它要在 shell 里反复试错
- 它要把工具调用、文件系统和长上下文串起来

这类能力非常接近真实软件工程，但训练起来很麻烦。数据要能执行，环境要能复现，任务不能太浅，reward 还得真的能区分成功和失败。

Tmax 的贡献是给出了一个简单、开放、可复现的 RL baseline。

作者先生成 terminal environments。生成过程用一个 taxonomy 控制任务分布，避免把数据生成退化成随手写 shell 题。这个 taxonomy 组合了三类信号：

- difficulty control
- personas
- verifier diversification

简单说，它要控制任务难度，让任务覆盖不同用户画像和工作方式，同时让 verifier 本身也足够多样。这样生成出来的任务才不会全部长成同一种浅层命令行练习。

然后，作者把这些环境用于 SFT 和 RL。RL 部分采用的是一个很朴素的 outcome-only recipe。模型最终有没有完成任务，由 verifier 给出结果，整套训练尽量少依赖复杂的过程奖励。

结果挺值得注意。

Tmax 使用 9B 参数模型，在 Terminal-Bench 2.0 上达到 **27%**。这个结果超过了之前一些更大的开放模型。论文还释放了数据、模型和代码，terminal dataset 的规模超过此前公开 terminal-agent datasets 的 **2.5 倍**。

这篇 paper 对我来说最有价值的地方，是它把 terminal agent 的训练问题落到了四个工程对象上：

1. task generation
2. verifier design
3. SFT warmup
4. outcome-only RL

这四件事都可以被复现、替换、扩展。

很多 agent 论文会把重点放在更复杂的规划框架、更炫的 multi-agent orchestration、更长的 prompt。Tmax 的路线更朴素：先把可执行任务和 verifier 做扎实，再用一个足够简单的 RL recipe 推模型。

这对 code agent 很重要。

代码能力的提升不只来自更大的 base model。很多时候，关键在于你能不能构造出足够真实、足够可验证、足够覆盖能力边界的环境。terminal agent 正好处在这个交叉点：它需要语言能力，也需要工具使用能力，还需要执行反馈。

今天第二篇是 **CLI-Universe: Towards Verifiable Task Synthesis Engine for Terminal Agents**。

这篇和 Tmax 很像一对组合拳。

CLI-Universe 关注的是 terminal-agent training data 怎么合成。它先从多维 capability taxonomy 里采样任务候选，再用 evidence-guided deep research 去真实技术材料里找 grounding，然后把任务实例化成 Dockerized environments。

最关键的是过滤。

CLI-Universe 会用 rubric-gated test construction、hint-conditional filtering、strict fail-to-pass checking 去筛任务。整个 pipeline 大约会丢掉三分之二的候选，只留下真正可执行、可验证、非平凡的任务。

最后，他们用 6,000 条高质量 trajectories fine-tune Qwen3-32B，在 Terminal-Bench 2.0 上达到 **33.4%**。

这篇的启发很直接：agent training data 的核心价值不在生成多少，而在验证留下来的样本到底有没有训练信号。

如果 task instruction 模糊，execution path 太浅，test 太脆，模型很容易学到表面模式。terminal agent 的数据集必须把环境、命令、测试、失败路径都做成一套闭环。

第三篇是 **PlanBench-XL: Evaluating Long-Horizon Planning of LLM Tool-Use Agents in Large-Scale Tool Ecosystems**。

它评估的是大规模工具生态里的 long-horizon planning。

PlanBench-XL 有 327 个 retail tasks，覆盖 1,665 个工具。agent 只能在 retrieval-limited visibility 下逐步找工具、调用工具、发现中间证据、推断 sub-goals，最后完成目标。

论文还加入了 blocking mechanism，用来模拟真实世界里的缺失工具、失败工具、干扰工具。结果很有意思：最强模型在没有 blocking 时能到 **51.90%**，但在最严重的 blocking 条件下跌到 **11.36%**。

这个结果说明，很多 agent 的问题不在于会不会调用工具，而在于工具路径断掉时能不能意识到、能不能恢复、能不能找到更长的替代路径。

把三篇放在一起，今天的信号很集中：

terminal agent 的核心问题正在变得更工程化。真正有价值的能力包含写命令，也包含训练环境是否可执行，verifier 是否可靠，RL recipe 是否可复现，benchmark 是否能暴露真实失败模式。

今天先读 `2606.23321`。

如果你关心 coding agents、Terminal-Bench、SFT/RL、post-training data，或者想知道 open model 怎么在 terminal agent 上追近 frontier，这篇值得拆。

## 今日 3 篇精选

### 1) Tmax: A simple recipe for terminal agents
- 链接: https://arxiv.org/abs/2606.23321
- 摘要速读: 用 taxonomy 生成 terminal environments，结合 difficulty control、personas、verifier diversification，再用 SFT 和 outcome-only RL 训练 9B terminal agent。
- 为什么重要: 它给出了一个开放、简单、可复现的 terminal-agent RL baseline，在 Terminal-Bench 2.0 上达到 27%。

### 2) CLI-Universe: Towards Verifiable Task Synthesis Engine for Terminal Agents
- 链接: https://arxiv.org/abs/2606.22883
- 摘要速读: 一个面向 terminal agents 的可验证任务合成引擎，用 Dockerized environments、rubric-gated tests 和 fail-to-pass checking 筛出高质量训练任务。
- 为什么重要: 它说明 terminal-agent 数据的关键是 verifier 和过滤质量。6K 高质量 trajectories fine-tune Qwen3-32B 后，在 Terminal-Bench 2.0 上达到 33.4%。

### 3) PlanBench-XL: Evaluating Long-Horizon Planning of LLM Tool-Use Agents in Large-Scale Tool Ecosystems
- 链接: https://arxiv.org/abs/2606.22388
- 摘要速读: 在 1,665 个工具组成的大规模生态里评估 tool-use agents 的长程规划、工具检索、证据发现和失败恢复。
- 为什么重要: 它暴露了 agent 在工具缺失、工具失败、工具干扰下的恢复能力短板。

## 一句话结论

今天最强的研究信号是，**terminal agent 的进展越来越依赖可执行环境、可靠 verifier 和可复现的 SFT/RL recipe。** `2606.23321` 值得先读。
