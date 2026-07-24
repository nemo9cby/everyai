---
title: "Paper Digest: 2026-07-24"
categories: [Paper Digest]
tags: [AI, Agents, Deep Research, Reinforcement Learning, Post-Training, Code Agents]
---

今天最值得看的 paper，我会选 **AREX: Towards a Recursively Self-Improving Agent for Deep Research**。

Deep Research agent 最难的部分，往往发生在搜索进行很久以后。

Agent 已经打开过许多页面，收集了相互矛盾的证据，也写出了一版看似完整的答案。此时继续增加搜索轮数，常常只会让 context 更拥挤。真正影响结果的，是 agent 能否回答三个问题：

1. 哪些结论已经被可靠证据支持？
2. 哪些约束仍然没有满足？
3. 下一次工具调用应该补哪一个缺口？

AREX 给出的核心方案，是一组 recursively self-improving research agents。它把工作过程拆成两个相互嵌套的 loop。

Inner research loop 负责搜索、阅读证据、构造 provisional answer。

Outer self-improvement loop 逐条检查任务约束，审计当前答案，标出 unresolved claims，再发起有明确目标的 follow-up research。

这个结构的价值在于，self-correction 获得了可操作的状态。Agent 手里保存的不再是一句宽泛的“请检查答案”，而是一份持续更新的 constraint ledger。哪些 claims 已验证，哪些证据冲突，哪些条件仍然缺失，都可以直接决定下一步 action。

## Learned context update

长轨迹还有一个现实问题：raw history 会持续增长。

当搜索记录、网页摘录、工具结果和中间答案全部留在 context 中，真正关键的证据容易被淹没。简单 summary 又可能丢失决定任务成败的约束。

AREX 训练了一个 autonomous context-update tool，把不断增长的 interaction history 压缩成 compact improvement state。这个 state 重点保留两类信息：

- verified evidence
- unresolved constraints

这个设计很像研究员整理工作笔记。读完一批材料后，笔记里需要留下可靠结论、证据出处、矛盾点和待查问题。网页访问顺序与大量重复内容可以被压缩。

论文强调，这个 context update 由 agent 自己学习完成，无需额外调用一个更强的外部模型。对于长任务，这同时影响质量与成本：agent 可以持续工作，又不必让 prompt 随着每轮搜索无限膨胀。

## Training recipe

AREX 的训练包含 verified synthetic tasks、high-quality trajectories、agentic mid-training 和 long-horizon reinforcement learning。

Long-horizon agent training 的 reward 很稀疏。一次 research episode 可能包含几十次工具调用，最终答案得分却只能告诉模型整条轨迹是否成功。真正决定成功的动作，可能只是中间找到的一条关键证据，或者某次及时放弃了错误方向。

作者因此强调两类关键步骤：

- decisive evidence acquisition
- correction of erroneous research directions

这相当于把 credit assignment 对准 research process 中信息增益最高的节点。对于 post-training，这比单纯拉长 rollout 更值得关注。Long horizon 会放大探索空间，也会放大错误积累。训练信号需要告诉 agent 哪些中间决策真正改变了最终结果。

作者训练了两个规模的模型：

- dense 4B
- 122B-A10B Mixture-of-Experts

它们在 BrowseComp、WideSearch、DeepSearchQA、Humanity's Last Exam，以及其他 reasoning 和 tool-use benchmarks 上显著超过同规模 baseline，并能与 activated parameters 更大的模型竞争。

## 对 agent 系统的启发

AREX 最有复用价值的部分，可以浓缩成四个工程对象：

1. **Constraint representation**：把复杂请求拆成可验证条件。
2. **Evidence state**：区分已验证、冲突和缺失的信息。
3. **Targeted follow-up**：每轮工具调用由具体缺口触发。
4. **Context update**：把长轨迹压缩成可继续行动的状态。

这四个对象让 self-improvement 获得了清晰的接口。Verifier 输出 unresolved constraints，context updater 保存当前研究状态，research policy 根据缺口选择下一步，最终答案再接受新一轮审计。

对于 code agent，也可以找到直接对应关系。任务约束可以是 failing tests、API contract 和 repository conventions；verified evidence 可以是通过的测试、调用链和复现日志；targeted follow-up 可以是读取特定模块或构造最小实验；context state 则保存已经排除的假设和剩余 failure modes。

今晚的深度分析值得继续追四个问题：

1. Context-update tool 的训练目标如何定义，怎样衡量压缩后有没有丢失关键约束。
2. Outer loop 的 constraint-wise verifier 使用哪些信号，错误审计会不会引导 agent 反复搜索。
3. Agentic mid-training 与 long-horizon RL 各自贡献多少，二者是否存在明显的阶段依赖。
4. 4B dense 与 122B-A10B MoE 在 research depth、tool efficiency 和 context update quality 上有哪些不同。

今天第二篇是 **Predictive Divergence Masks for LLM RL**。

PPO-style trust-region mask 通常用 sampled-token importance ratio 同时判断两个问题：policy 是否已经偏离 behavior policy 太远，以及下一步 gradient 会不会让它偏得更远。

DPPO 用 probability divergence 改进了第一个判断，direction criterion 仍然来自 sampled ratio。论文指出，这两个判断可能出现符号冲突。某个 token 的 ratio 看起来正在远离 1，完整分布的 divergence 却可能下降。

作者推导了 predictive divergence mask，直接预测下一次 policy-gradient step 会让同一个 divergence 增加还是减少。对于 discrete softmax policy，这个预测存在 closed-form solution。考虑到 production rollout engine 常常只返回 top-K vocabulary，论文还给出了两个轻量 estimator。

它对 LLM RL 很实用，因为 mask 决定哪些 token 真正贡献 gradient。方向判断若与实际 divergence change 不一致，训练会错误丢弃有效更新，或者保留破坏 trust region 的更新。

第三篇是 **Sample-Efficient Learning from Agent Experience**。

论文研究一个很具体的问题：agent 把过去的 trial-and-error history 放进 context 后，能力会提高；一旦 history 被拿走，这部分提升也会消失。如何把 experience 带来的 improvement 写进模型参数？

作者提出 Experience Distillation。它先利用 interaction history 形成 in-context improvement，再把这种改进蒸馏进 weights，整个训练过程不需要增加 environment interaction。

在 749 个 software-engineering tasks 和六个 text-adventure games 上，它保留了至少 64.8% 的 in-context gain。直接拿收集到的 experience 做 SFT，只恢复了 3.8%。与 classical RL baseline 相比，它用至少少 9.6 倍的 environment samples 达到相近表现。

这里最重要的信号是 trajectory 的训练目标。Interaction log 本身只记录 agent 看到了什么、做了什么；完整 context 对后续 policy 造成的改变，包含了更丰富的学习信息。Experience Distillation 试图保留后者。

## 今日 3 篇精选

### 1) AREX: Towards a Recursively Self-Improving Agent for Deep Research
- 链接: https://arxiv.org/abs/2607.21461
- 摘要速读: 用 inner research loop 和 constraint-wise outer audit 反复改进答案，并训练 context-update tool 保存 verified evidence 与 unresolved constraints。
- 为什么重要: 它把 long-horizon agent 的 context compression、verification、agentic mid-training 和 RL 放进了一条完整训练链路。

### 2) Predictive Divergence Masks for LLM RL
- 链接: https://arxiv.org/abs/2607.10848
- 摘要速读: 预测下一步 policy-gradient update 对 probability divergence 的真实方向，让 trust-region proximity 与 direction criterion 使用同一度量。
- 为什么重要: 它直接影响 off-policy LLM RL 中哪些 token 被 mask，且提供了适配 production top-K logits 的 estimator。

### 3) Sample-Efficient Learning from Agent Experience
- 链接: https://arxiv.org/abs/2607.21051
- 摘要速读: 把 agent 依赖 interaction history 获得的 in-context improvement 蒸馏进模型参数，无需额外环境交互。
- 为什么重要: 在 software-engineering tasks 上保留至少 64.8% 的 experience gain，直接 SFT 只保留 3.8%。

## 一句话结论

今天最强的信号是：**长轨迹本身不等于有效经验。Agent 需要持续保存已验证证据和未解决约束，训练信号也要对准真正改变研究方向的关键步骤。**
