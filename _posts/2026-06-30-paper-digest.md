---
title: "Paper Digest: 2026-06-30"
categories: [Paper Digest]
tags: [AI, Coding Agents, Post-training, RL, SWE-bench, Agent Infrastructure]
---

今天最值得看的 paper，我会选 **Dockerless: Environment-Free Program Verifier for Coding Agents**。

它切中的问题非常实际：coding agent 的 post-training，很多时候卡在 verifier 上。

如果要训练一个能修 repo issue 的 agent，你需要知道一条 trajectory 或一个 patch 到底好不好。最标准的办法是跑测试。问题是，每个 repo 都有自己的依赖、环境、版本、数据库、系统包、隐藏假设。于是 verifier 很快变成 Docker farm。

这件事很贵，也很脆。

Dockerless 的核心想法是：能不能不真正执行 patch，也能判断 patch correctness？

它提出了一个 environment-free agentic patch verifier。它不靠简单匹配 reference patch，也不跑 repo-specific Docker environment，而是让 verifier agent 去探索 repository，收集证据，再判断 candidate patch 是否解决了问题。

这听起来像一个小工程优化，但其实会影响整条训练链路。

在 coding-agent post-training 里，verifier 至少有两个关键用途：

- SFT 阶段：筛选哪些 trajectory 值得学
- RL 阶段：给 generated patch 提供 reward

如果 verifier 必须跑完整环境，那么扩数据、扩任务、扩模型都会被执行成本限制。Dockerless 试图把这个瓶颈拆掉。

结果也有意思。

论文报告 Dockerless 在 verifier evaluation benchmark 上，比最强 open-source verifier 高 **14.3 AUC points**。

更关键的是，他们把 Dockerless 同时用作 SFT trajectory filter 和 RL reward，形成一条 fully environment-free post-training pipeline。最后模型在三个 SWE-bench 变体上达到：

- SWE-bench Verified: **62.0%**
- SWE-bench Multilingual: **50.0%**
- SWE-bench Pro: **35.2%**

相对 Qwen3.5-9B baseline，分别提升 **2.4 / 8.7 / 2.9 points**，并且接近 environment-based post-training。

这篇 paper 对我最大的价值，是它把 code-agent training 的一个底层问题讲清楚了：reward 并不免费。

很多论文会把 RL 写成一个比较抽象的过程，好像只要有 benchmark，就可以很自然地产生 reward。软件工程任务里没那么简单。一个 patch 的 correctness 往往依赖复杂环境，依赖隐藏测试，依赖跨文件语义，也依赖 issue 的真实意图。

Dockerless 给了一个很值得追的方向：用 agentic evidence gathering 来替代一部分 execution-based verification。

它不一定能覆盖所有场景。对于需要真实运行、性能回归、并发 bug、系统集成的任务，执行环境仍然很重要。但如果它能稳定过滤大规模 SFT/RL 数据，哪怕只覆盖一部分 repo-level task，也会让 code-agent post-training 的成本结构变得更好。

今天第二篇是 **Agentic Abstention: Do Agents Know When to Stop Instead of Act?**

这篇研究的是 agent 什么时候应该停止行动。

普通 LLM abstention 往往是一轮问答：知道就答，不知道就拒绝。Agentic abstention 更难，因为 agent 可以继续查资料、点网页、跑 terminal、调用工具。很多任务一开始看起来可行，直到环境交互之后才发现目标无法满足。

论文把这件事定义成 sequential decision problem：agent 在每一步都要决定 answer、abstain，还是继续收集信息。

他们评估了 13 个 LLM-as-agent systems 和 2 个 agent scaffolds，覆盖 web shopping、terminal environments、question answering，超过 28,000 个任务。

核心发现不是 agent 会不会 abstain，而是 **什么时候 abstain**。

有些 agent 明明应该停，却一直继续尝试。有些 agent 最后会停，但已经浪费了很多无意义交互。论文还提出 CONVOLVE，把完整 interaction trajectories 蒸馏成 reusable stopping rules，在 WebShop 上把 Llama-3.3-70B 的 timely recall 从 **26.7** 提高到 **57.4**，而且不需要更新模型参数。

这对生产 agent 很重要。一个可靠 agent 不只需要更会调用工具，还需要知道继续调用工具已经没有边际收益。

第三篇是 **When AI Reviews Its Own Code: Recursive Self-Training Collapse in Code LLMs**。

这篇和 Dockerless 很搭。

它讨论一个未来会越来越常见的问题：AI 写的代码进入真实仓库，之后这些仓库又被拿来训练 code LLM。这样就形成了 repository-scale recursive self-training loop。

作者比较了三种 recursive fine-tuning regime：

1. no review
2. Human-gate review，用 compilation、static checks 这类 model-independent filters
3. AI-self-gate review，用 code LLM 自己的 perplexity、binary self-score 这类信号

结果很警醒。

no review collapse 最快。Human-gate 能减缓 collapse。AI-self-gate 一开始看起来不错，但后面可能进入 rubber-stamp regime：acceptance score 越来越高，benchmark correctness 反而下降。

这其实是在提醒 code model 训练团队：如果 verifier 和 generator 强耦合，系统很容易自我确认。模型越相信自己的输出，训练数据越被自己的偏差污染。

把今天三篇放在一起，主题很集中：

**code agent 的上限不只取决于 base model，还取决于 verifier、stopping policy、data gate 这些运行时和训练时基础设施。**

Dockerless 说 verifier 成本会限制 RL/post-training。Agentic Abstention 说 agent 需要知道何时停止。Self-training collapse 说代码数据管道需要独立验证信号。

如果你关心 coding agents、SWE-bench、SFT/RL、post-training infrastructure，今天先读 `2606.28436`。

## 今日 3 篇精选

### 1) Dockerless: Environment-Free Program Verifier for Coding Agents
- 链接: https://arxiv.org/abs/2606.28436
- 摘要速读: 提出 environment-free agentic patch verifier，不运行 per-repo Docker/test environment，而是通过 repository exploration 收集证据来判断 patch correctness。
- 为什么重要: 它把 verifier 用作 SFT trajectory filter 和 RL reward，让 code-agent post-training 有机会摆脱一部分环境执行成本。

### 2) Agentic Abstention: Do Agents Know When to Stop Instead of Act?
- 链接: https://arxiv.org/abs/2606.28733
- 摘要速读: 把 agent abstention 定义成 sequential decision problem，评估 13 个 LLM-as-agent systems 在 web、terminal、QA 任务中何时应该停止。
- 为什么重要: 长程 agent 不能只会继续调用工具，还要知道目标已经不可达，继续行动只是在浪费 step budget。

### 3) When AI Reviews Its Own Code: Recursive Self-Training Collapse in Code LLMs
- 链接: https://arxiv.org/abs/2606.28438
- 摘要速读: 研究 code LLM 在 AI-generated code 进入训练数据后的 recursive self-training collapse，并比较 no review、human-gate、AI-self-gate 三种过滤机制。
- 为什么重要: 它说明 code model 训练需要独立验证信号，模型自己的 self-score 可能在后期变成 rubber stamp。

## 一句话结论

今天最强的研究信号是，**coding agent 的 post-training 正在被 verifier infrastructure 重新定义：reward 怎么来、数据怎么筛、agent 什么时候停，会直接决定模型能力能不能真正变成 repo-level success。** `2606.28436` 值得先读。
