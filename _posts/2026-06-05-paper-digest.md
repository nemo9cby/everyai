---
title: "Paper Digest: 2026-06-05"
categories: [Paper Digest]
tags: [AI, Code Generation, Post-Training, Reinforcement Learning, Agents, LoRA]
---

今天最值得看的 paper，我会给 **Combinatorial Synthesis: Scaling Code RLVR via Atomic Decomposition and Recombination**。

这篇 paper 关心一个很硬的 code model 问题：RLVR 要怎么继续扩大。

RLVR 的吸引力很清楚。代码任务天然适合 verifiable rewards，只要 test case 或 checker 设计得好，模型生成的答案能不能跑过，系统可以自动判断。过去一年很多 code capability 的提升，都和这种可验证训练信号有关。

真正卡住的地方在数据。简单题太容易，模型学不到东西；太难或太怪的题又容易变成噪声；只有那些刚好贴近模型能力边界的任务，训练价值最高。问题是，这种题很少。很多 synthetic data 方法会围绕 seed problem 做扩写、改写、换皮，但生成规模上去以后，任务的新颖性和难度未必同步上去。

这篇提出的 ADR，Atomic Decomposition and Recombination，思路是把代码任务拆成 atomic elements，再受控地重新组合。

它的直觉很像搭积木。一个 code problem 里通常包含多个能力单元，比如数据结构选择、边界条件、状态转移、工具调用、I/O 约束、复杂度限制。ADR 先把这些元素拆出来，再用组合方式生成新的可验证任务。这样合成出来的题有机会同时满足三个条件：足够新，足够难，还能自动验证。

这比普通 prompt mutation 更有训练意义。prompt mutation 很容易只改变表述，核心能力点没有变；ADR 直接操作问题结构，能把不同能力单元重新交叉，制造出模型没有见过、但仍然有明确 reward 的任务。

论文报告说，ADR 在 originality、difficulty、diversity 和 test quality 上都超过已有 baseline，并且在 RLVR 训练后带来更强的代码能力提升。下游覆盖 algorithmic programming、tool usage 和 data science。对我来说，这几个 domain 的组合很重要，因为它说明作者把测试面扩到了更接近真实 code assistant 的任务集合。

这篇的价值在于，它把 code RLVR 的核心瓶颈往上游推了一层。很多讨论会集中在 reward model、policy optimization、rollout strategy、pass@k 指标，但如果训练任务本身质量不够，后面的优化再精细也很难持续吃到能力增长。ADR 直接问：好任务从哪里来？

对做 post-training 的团队，这个问题非常实际。真正稀缺的是更多“刚好有训练价值”的题。它们要有清晰可验的答案，要覆盖真实能力点，要能随着模型变强继续提高难度。ADR 给了一个可操作的方向：把任务拆成能力原子，再通过组合制造新的训练压力。

今天第二篇是 **Code2LoRA: Hypernetwork-Generated Adapters for Code Language Models under Software Evolution**。

它处理的是 repo context 的成本问题。代码模型要理解 imports、APIs、project conventions，通常需要 repository-level context。现在常见做法有两种：一种是 RAG 或依赖分析，把相关文件塞进 prompt；另一种是 per-repository fine-tuning 或 LoRA。前者会吃 inference tokens，后者在 repo 数量变大、代码不断变化时很难维护。

Code2LoRA 走的是生成 adapter 的路线。它用 hypernetwork 根据仓库生成 LoRA adapter，把 repo knowledge 压进参数里，推理时没有额外 token 开销。静态场景里，Code2LoRA-Static 把一个 repo snapshot 转成 adapter；演进场景里，Code2LoRA-Evo 用 GRU hidden state 随 code diff 更新 adapter。

这篇还做了 RepoPeftBench，包含 604 个 Python repositories。static track 有 40K training 和 12K test assertion-completion tasks；evolution track 有 215K commit-derived training 和 87K commit-derived test tasks。这个 benchmark 本身就值得记一下，因为它把“repo-aware code model”拆成了 stable codebase 和 evolving codebase 两个更真实的评估面。

第三篇是 **Rethinking Continual Experience Internalization for Self-Evolving LLM Agents**。

它讨论 agent 如何把过去交互中的经验变成参数能力。这个方向很诱人，因为长期运行的 agent 会积累大量轨迹、失败、修复、工具调用策略。如果这些经验只能留在 context 或 memory 里，成本会越来越高；如果能变成模型能力，系统才有持续进步的可能。

论文指出，多轮 experience internalization 可能带来 progressive capability collapse。作者从三个维度拆这个问题：principle-level experience 比 instance-level trace 更耐用；step-wise injection 比 global injection 更适合长程工具调用；off-policy context-distillation on high-quality teacher trajectories 比 on-policy student trajectories 更稳定。

这个结论对 agent post-training 很有提醒意义。让 agent 学自己的经验，听起来很自然，但 student 自己产生的 flawed states 会把训练信号带偏。更稳的路线可能是先把经验抽象成原则，在决策步骤上注入，再用高质量 teacher trajectories 做 distillation。

三篇放在一起看，今天的信号很清楚：code-agent 进步越来越依赖“训练材料和经验怎么被内化”。Code RLVR 要更好的 verifiable task supply，repo-aware coding 要把项目知识压进可更新 adapter，self-evolving agent 要避免多轮经验学习把自己训塌。

所以今天先读 `2605.31058`。如果你关心 LLM code capability、RLVR、synthetic data、verifiable rewards，或者 post-training 里“数据到底怎么造才有用”，这篇值得拆。

## 今日 3 篇精选

### 1) Combinatorial Synthesis: Scaling Code RLVR via Atomic Decomposition and Recombination
- 链接: https://arxiv.org/abs/2605.31058
- 摘要速读: 提出 ADR，把代码任务拆成 atomic elements 后重新组合，用来合成更难、更原创、仍可验证的 code RLVR 训练任务。
- 为什么重要: Code RLVR 的瓶颈不只是优化算法，还有高质量 verifiable tasks 的持续供给。

### 2) Code2LoRA: Hypernetwork-Generated Adapters for Code Language Models under Software Evolution
- 链接: https://arxiv.org/abs/2606.06492
- 摘要速读: 用 hypernetwork 为仓库生成 LoRA adapter，把 repo knowledge 压进参数，并支持随 code diff 更新。
- 为什么重要: 它给 repo-aware code model 提供了一条长 context 和 per-repo fine-tuning 之外的路线。

### 3) Rethinking Continual Experience Internalization for Self-Evolving LLM Agents
- 链接: https://arxiv.org/abs/2606.04703
- 摘要速读: 研究 agent 如何把过去经验内化成参数能力，并指出多轮 internalization 会出现 capability collapse。
- 为什么重要: Self-evolving agents 需要稳定吸收经验，不能让自己的错误轨迹污染后续训练。

## 一句话结论

今天最强的研究信号是：**code RLVR 和 code agents 的上游问题正在变得更重要，真正稀缺的是可验证、可组合、可持续升级的训练任务和经验。** `2605.31058` 值得先读。
