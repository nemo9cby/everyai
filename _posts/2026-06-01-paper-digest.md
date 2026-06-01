---
title: "Paper Digest: 2026-06-01"
categories: [Paper Digest]
tags: [AI, Agents, Search Agents, Coding Agents, Post-Training, Reinforcement Learning]
---

今天最值得看的 paper，我会给 **GrepSeek: Training Search Agents for Direct Corpus Interaction**。

这篇 paper 讲 search agent，但它抓住了一个更大的问题：如果 agent 可以直接操作环境，检索就不一定要被压缩成一次向量查询。

GrepSeek 的选择很大胆：让 search agent 把语料库当成一个可执行环境，直接发 shell command 查找证据。模型可以 grep、过滤、组合、继续搜索，像一个会用命令行的研究助理一样在 corpus 里工作。

这件事有两个好处。

第一，search trajectory 变得可观察。你能看到 agent 用了什么命令，怎么缩小范围，哪些证据被找到，哪里出现了 lexical mismatch。

第二，search behavior 可以被训练。作者用两阶段 pipeline 解决直接 RL 不稳定的问题：先用 answer-aware Tutor 和 answer-blind Planner 构造 verified、causally grounded 的冷启动搜索轨迹，再用 **GRPO** 让 policy 在语料库环境里继续优化搜索行为。

为了让这种 direct corpus interaction 能跑在大语料上，GrepSeek 还做了一个 sharded-parallel execution engine。它把 shell-based retrieval 加速到最高 7.6x，同时保持和顺序执行 byte-exact 等价。这个细节很工程，因为 command semantics 一旦变了，训练和评估信号都会变脏。

实验上，GrepSeek 在 7 个 open-domain QA benchmarks 上拿到最强的整体 token-level F1 和 Exact Match。更重要的是，它证明了一件对 agent 训练很有启发的事：有些检索能力可以通过和 corpus 的直接互动学出来，而不必完全依赖预先建好的 embedding index。

这对 coding agent 也有明显借鉴意义。真实 repo 任务里，agent 经常需要用 `rg`、`grep`、`find`、`sed`、test output 和 log 来定位问题。很多有价值的行为并不发生在一次 semantic search 里，而发生在一串可执行命令和环境反馈之间。

GrepSeek 给出的范式是：把环境暴露出来，把轨迹造出来，把 reward 做实，再用 RL 去优化 agent 如何行动。

今天另外两篇也值得一起看。

**Combinatorial Synthesis** 处理 code RLVR 的数据瓶颈。它提出 Atomic Decomposition and Recombination，把代码任务拆成 atomic elements，再受控重组出新的、困难的、可验证的任务。核心价值在于生成 near-frontier code tasks。RLVR 要继续扩大，真正稀缺的是有训练价值的 verifiable tasks。

**Mellum2 Technical Report** 则是软件工程模型方向的技术报告。Mellum 2 是一个 open-weight 12B MoE，单 token 激活 2.5B 参数，面向 code generation、editing、debugging、tool use、agentic coding 和 conversational programming。报告里有 architecture、10.6T token curriculum、Muon + FP8、128K context、SFT + RLVR 等细节，适合做 code-specialized foundation model recipe 的参考。

三篇放在一起看，今天的信号很清楚：agent 和 code model 的进展越来越依赖可验证的训练基础设施。search agent 需要可执行 corpus，code RLVR 需要困难且可测的任务，software-engineering foundation model 需要清晰的数据和 post-training recipe。

所以今天先读 `2605.29307`。如果你关心 coding agent、retrieval agent、GRPO、或者想训练更会使用命令行环境的模型，这篇最值得拆。

## 今日 3 篇精选

### 1) GrepSeek: Training Search Agents for Direct Corpus Interaction
- 链接: https://arxiv.org/abs/2605.29307
- 摘要速读: 训练 search agent 直接用 shell commands 和 corpus 互动，先用 Tutor/Planner 构造冷启动轨迹，再用 GRPO 优化搜索行为。
- 为什么重要: 它把 retrieval 变成可观察、可执行、可训练的 agent-environment interaction。

### 2) Combinatorial Synthesis: Scaling Code RLVR via Atomic Decomposition and Recombination
- 链接: https://arxiv.org/abs/2605.31058
- 摘要速读: 通过 atomic decomposition 和 recombination 生成新颖、困难、可验证的代码任务，用于扩展 code RLVR。
- 为什么重要: Code RLVR 的瓶颈常常是高质量 verifiable tasks，这篇给了一个更可控的数据生成方向。

### 3) Mellum2 Technical Report
- 链接: https://arxiv.org/abs/2605.31268
- 摘要速读: 一个面向软件工程的 open-weight 12B MoE 模型报告，覆盖架构、数据 curriculum、长上下文扩展、SFT 和 RLVR。
- 为什么重要: 它提供了 code-specialized foundation model 的完整 recipe，而不只是 leaderboard 数字。

## 一句话结论
今天最强的研究信号是：**agent 训练正在围绕可执行环境和可验证任务重新组织。** `2605.29307` 值得先读。
