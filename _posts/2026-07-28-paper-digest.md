---
title: "Paper Digest: 2026-07-28"
categories: [Paper Digest]
tags: [AI, Kimi K3, Agentic RL, Mixture of Experts, Code Agents, Post-Training]
---

今天最值得看的 paper，我会选 **Kimi K3: Open Frontier Intelligence**。

这是 Moonshot AI 对 Kimi K3 的完整技术报告。模型总参数 2.8T，每个 token 激活 104B 参数，原生支持 vision，并提供 1M-token context window。模型权重已经开放。

这些数字足够醒目，但真正值得读的是报告里几条互相咬合的技术线：

- Kimi Delta Attention 与 Attention Residuals
- 896 个 routed experts 中每个 token 激活 16 个
- 约 2.5 倍于 Kimi K2 的整体 scaling efficiency
- 覆盖 general、coding 与 agentic domains 的 reinforcement learning
- million-token agentic RL 与 persistent rollout sandbox states
- balanced expert-parallel training、memory management 与 deployment optimization

K3 展示的是一整套 frontier model engineering。Architecture、training system、post-training 和 agent runtime 需要共同工作，任意一层的瓶颈都会限制最终能力。

## 1M context 怎样进入 RL

长 context 的 pretraining 已经很难。进入 agentic RL 后，问题会继续放大。

Agent trajectory 包含模型输出、工具调用、环境 observation、文件变化、失败尝试和跨轮状态。Context 越长，rollout worker 需要维护的状态越大，采样时间差异也越明显。Learner 持续更新 policy，环境却可能仍在执行一条很久以前开始的 trajectory。

K3 报告特别提到 million-token agentic RL，以及 persistent rollout and sandbox states。这意味着 sandbox 生命周期已经进入训练系统的核心路径。

一个长任务可能经历：

1. 初始化代码仓库或工作环境
2. 多轮读取、修改和执行
3. 保存工具输出与环境状态
4. 在 policy 更新期间继续 rollout
5. 对最终结果与中间行为计算 reward

如果环境每次重建，成本会很高，任务连续性也会丢失。若环境长期保留，系统又需要解决隔离、故障恢复、存储、版本一致性和资源回收。

长 context 在这里是一种系统负担，也是一种训练数据结构。模型的 planning horizon、sandbox 的生命周期和 learner 的更新节奏会彼此影响。

## Stable LatentMoE 与 expert balance

K3 有 896 个 routed experts，每个 token 激活其中 16 个。如此大的 expert pool 可以提供更高容量，同时让单次 forward 保持有限计算量。

Agentic workload 会让 MoE routing 更复杂。Coding、search、general reasoning、vision 与 tool-use 的 token distribution 差异很大。长 trajectory 内部也会经历明显的阶段变化，比如从理解需求，到搜索文件，再到写代码与解释结果。

论文将 Stable LatentMoE、balanced expert-parallel training 和 memory management 放在同一个系统设计里。这里的关键约束包括：

- experts 的 token load 能否保持平衡
- routing 是否在不同 domain 间形成稳定 specialization
- expert parallel communication 会不会吞掉稀疏计算收益
- long-context activation 与 KV state 如何影响显存
- RL rollout 的任务异质性是否加剧负载波动

模型层面的 sparse activation，需要系统层面的 data placement 与 communication schedule 来兑现效率收益。

## Post-training 覆盖多个 capability domains

K3 的 post-training 同时覆盖 general、agentic 和 coding domains，并支持多个 reasoning-effort levels。

多 domain RL 很容易出现 reward scale、数据比例和能力干扰问题。Coding task 通常有 executable verifier，agent task 可能依赖环境终态，general reasoning 则常使用 rule-based 或 model-based evaluation。不同 reward 的噪声、密度与可验证性差别很大。

多 reasoning-effort levels 还引入了额外控制目标。模型要学会在简单任务上少花 token，在复杂任务上延长计算，同时保持答案质量与成本之间的稳定关系。

这部分值得追问几件事：

1. 不同 domain 的 rollout 如何混合与调度。
2. Reward scale 如何校准，避免某类任务主导更新。
3. Coding 与 agentic trajectory 是否共享 replay 或过滤规则。
4. Reasoning effort 由显式 control token、训练分布还是 reward 共同塑造。
5. Million-token trajectory 如何计算 advantage，并控制 policy lag。

技术报告给出了大图，真正可复现的价值会取决于这些训练细节能开放到什么程度。

## 为什么今天选它

今天还有几篇更聚焦的 agent 与 post-training 论文。K3 的优势在于，它把 frontier model 的多个难点放进一个实际系统。

对研究者来说，单独研究 attention、MoE、RL 或 agent harness 都很自然。生产级 frontier model 需要同时回答这些问题：

- Architecture 能否提高 scaling efficiency
- Distributed training 能否稳定承载 sparse model
- Post-training 能否塑造 coding 与 long-horizon agent ability
- Rollout infrastructure 能否维护 persistent environment state
- Serving system 能否把 1M context 与 MoE capacity 交付给用户

K3 给出了一份少见的全链路样本。即使最终能力仍落后于最强 proprietary models，它对 open-weight ecosystem 的价值也很直接：研究者可以把报告里的 training claims 与实际 weights、inference behavior 和 downstream agent performance 对照起来。

## 另外两篇

第二篇是 **StateAct: Program State, before Pixels, for Long-Horizon Computer-Use Agents**。

StateAct 让主 agent 直接操作 files、application backends 和 DOM 等 program state，仅把确实需要视觉交互的 subgoals 交给 GUI subagent。在 OSWorld 2.0 的 108 个任务里，只有 28 个使用 GUI subagent，这些交互只占主 agent steps 的 1.1%。

它还增加了独立 finish gate，检查结果是否缺失、未保存或写错路径。Claude Opus 4.8 的 binary success 从 20.6% 提高到 26.9%，partial success 从 54.8% 提高到 61.6%，单任务成本约为 screenshot-only harness 的九分之一。

一个重要 ablation 是，完全 code-only 的版本只有 45.9% partial success，低于 screenshot baseline 的 54.8%。Hybrid routing 才是最有效的配置。

第三篇是 **The Physics of Multi-Turn Long-Horizon Planning**。

这篇论文在受控环境中研究 planning ability 如何受到 pretraining、GRPO、single-teacher on-policy distillation 和 multi-teacher distillation 的影响。

它观察到：

- 显式建模 state transition 的 CoT 有助于 long-horizon generalization
- atomic skills 本身无法带来 compositional planning
- 少量高质量 long-horizon data 就能产生帮助
- suboptimal trajectory 的错误会沿 horizon 放大
- 在低质量数据与长 horizon 下，on-policy distillation 的有效区间比 GRPO 更宽
- compatible planning patterns 可以通过 multi-teacher distillation 整合，冲突 patterns 会造成明显 interference

对于 agentic post-training，这些结论都能转化成具体实验假设。

## 今日 3 篇精选

### 1) Kimi K3: Open Frontier Intelligence
- 链接: https://arxiv.org/abs/2607.24653
- 摘要速读: 2.8T MoE、104B activated parameters、1M context，并把 coding/agentic RL、balanced expert parallelism 与 persistent rollout sandboxes 放进同一套 frontier stack。
- 为什么重要: 它提供了一份从 architecture、distributed training、post-training 到 agent runtime 的开放技术样本。

### 2) StateAct: Program State, before Pixels, for Long-Horizon Computer-Use Agents
- 链接: https://arxiv.org/abs/2607.22798
- 摘要速读: 主 agent 直接操作 program state，少量视觉任务交给 GUI subagent，再用独立 finish gate 验证最终 artifact。
- 为什么重要: State-grounded hybrid harness 同时提高成功率并显著降低成本，纯 code 或纯 screenshot 都达不到相同效果。

### 3) The Physics of Multi-Turn Long-Horizon Planning
- 链接: https://arxiv.org/abs/2607.24720
- 摘要速读: 在受控环境中比较 pretraining、GRPO、on-policy distillation 与 multi-teacher distillation 如何塑造 planning patterns 和 task knowledge。
- 为什么重要: 它给出了何时 GRPO 有效、何时 teacher signal 更稳定，以及多 teacher 能力整合何时发生 interference 的实验框架。

## 一句话结论

今天最强的信号是：**Frontier agent capability 取决于 model architecture、post-training、distributed execution、persistent environment state 与 verification harness 的共同设计。**
