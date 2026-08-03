---
title: "Paper Digest: 2026-08-03"
categories: [Paper Digest]
tags: [AI, Reinforcement Learning, RLVR, Post-Training, Self-Play, Code Agents]
---

今天最值得看的 paper，我会选 **From RLVR to RLSVR: Task Transformation Induces Self-Verifiable Rewards for Open-Ended LLM Self-Improvement**。

它问了一个很实际的问题：RLVR 在数学和代码上很好用，因为答案、单元测试或编译器可以给出确定的 reward。到了 summarization、creative writing 这类开放任务，正确答案消失了，训练通常又要依赖 human preference、reward model、rubric executor 或 LLM judge。

RLSVR 提供了另一种 reward construction。它先改造任务，让环境主动注入一个可验证的 latent variable，再让模型完成原任务并推断这个 latent variable。最终 reward 检查环境已经知道的答案，目标能力则决定模型能否在新游戏里获胜。

论文给出的实现叫 SpyRL，灵感来自“谁是卧底”。这个名字听起来像一层 game wrapper，实际包含一套颇完整的 multi-agent self-play 与 GRPO 训练设计。

## 把开放任务变成信息不对称游戏

一次 SpyRL episode 有五个 players。

四个 civilians 收到完整输入，一个 spy 只收到经过 degradation 的输入。所有人仍要完成相同的 target task。例如：

- summarization 中，civilian 阅读完整文档，spy 的文档会被连续遮掉一部分内容
- creative writing 中，不同角色获得的信息完整度不同，但仍需写出同一类故事
- mathematical reasoning 中，信息缺失会影响题目构造或解题质量

五个 outputs 随后公开。每个 player 根据这些 outputs 投票，猜谁拿到了残缺信息。

Spy identity 在 episode 开始时由环境随机指定，因此 detection reward 可以精确计算：猜中为 1，猜错为 0。整个过程不需要 human label、reward model 或 external judge。

这里有一个关键约束。Information-degradation operator 必须删除完成任务所需的信息，同时保留主题、长度和表面风格。若 spy output 带有明显格式特征，detector 会学会 shortcut；若被删信息无关紧要，这局游戏也不会训练目标能力。

论文在 summarization 上比较 20% 与 40% continuous span masking，五个 benchmarks 的结果很接近。作者据此认为 degradation operator 不需要精细调参，只要产生有意义且不过度的能力差异即可。这个结论值得在更多任务上复现，因为它决定 RLSVR 能否成为通用 recipe。

## 投票怎样变成 performer reward

Detection stage 的 reward 很直接，performing stage 更微妙。

每个 output 收到的 suspicion votes 越多，说明它越像由信息不足的 spy 生成，performer reward 也越低。Spy 希望在 civilians 中隐藏自己，civilians 则希望产出足够完整、自然且可信的结果，避免被误认成 spy。

论文用 100 局 WritingPrompts 和 GovReport 进行验证，同时让 GPT-4o 对每局五个 outputs 做质量排序。收到更多 votes 的 outputs，平均质量排名也更差。这说明 vote signal 与外部质量评价存在正相关。

这个 proxy 依然需要谨慎理解。环境能够精确验证 spy identity，votes 是否完整代表真实 task quality 仍是经验问题。模型可能学到 detectors 特有的偏好，也可能发现 game-specific strategy。作者用 GPT-4o ranking、ROUGE 和 blind human evaluation 交叉验证，证据比单一 LLM judge 更扎实，尚不足以证明任意开放任务都能获得同样可靠的 alignment。

## Role-Advantage Estimation 是不可省的细节

Spy 天生处于信息劣势。直接把 raw performing rewards 混在一起做 policy optimization，会把角色难度当成 policy quality。

SpyRL 分别维护 spy 与 civilian 的 exponential-moving-average baselines。每条 rollout 只和自己角色的预期表现比较，这个机制称为 Role-Advantage Estimation（RAE）。

消融结果非常醒目：Qwen3-4B 在七个 reasoning benchmarks 上的 base average 是 41.4。加入完整 SpyRL 后升到 50.4；移除 RAE 后跌到 37.5，低于 base model。

也就是说，role calibration 直接决定训练方向。对于任何 asymmetric self-play，reward 的绝对大小通常混合了 policy competence、role difficulty 与 environment state。把这些因素拆开，是训练成立的前提。

## Performer 与 detector 必须一起学习

SpyRL 交替优化 performing stage 和 detection stage。

冻结 detector，只训练 performer，Math500 accuracy 会先提升，随后在约 72% 附近停滞。Detector 无法识别更强 outputs 之间的差异后，votes 逐渐失去质量信号。只训练 detector 的收益更小；拿掉 spy 所提供的信息不对称，训练也很快进入平台期。

完整的 coupled optimization 将 Math500 从 68.2% 提高到 79.5%。训练过程根据 detection accuracy 和 abstention rate 的 moving averages 切换 active phase，并设 minimum dwell time，避免频繁切换造成震荡。

这个设计有点像持续更新的 reward model，只是 evaluator 的 ground truth 来自环境身份，训练信号由群体投票产生。Performer 变强后，detector 也要提高分辨率；detector 变强后，又给 performer 提供更严格的竞争压力。

Group size 也会改变训练难度。Player 数量从 3 增加到 5 时，五个 reasoning benchmarks 的平均 gain 从 5.5 增加到 9.3；继续扩到 6 或 8 后边际收益明显下降。论文最终使用五人一局，在 game complexity 与 rollout cost 之间取了一个经验平衡点。

## 结果怎样看

在开放任务上，Qwen3-8B + SpyRL 相对 untrained backbone 的平均 A/B win rate 为：

- summarization：75.4%
- creative writing：77.3%

Summarization 的五个 benchmarks 上，两个 model sizes 的 ROUGE-L 都超过 R-Zero、Absolute Zero 与 base model。Creative writing 的 blind human evaluation 由 10 名 PhD students 完成，共 400 个 prompts。SpyRL 在 WritingPrompts 上相对 Qwen3-4B、R-Zero 和 Absolute Zero 的 overall win rates 分别达到 80.0%、78.5% 与 74.0%，WritingBench 上也保持类似优势。

在已有 verifiable rewards 的 reasoning tasks 上，SpyRL 仍然报告最强结果。七个 benchmarks 的平均提升为：

- Qwen3-4B：+8.97 percentage points
- Qwen3-8B：+6.16 percentage points

更细的数据包括 Qwen3-4B 的 Math500 从 68.2 提高到 79.5，AIME25 从 6.7 提高到 20.0，GPQA-Diamond 从 26.3 提高到 41.3。

论文也与 rubric-as-reward 做了同一 GRPO 框架下的比较。Qwen3.5-27B 与 GPT-4o 作为 rubric executors，额外 verifier cost 约为 200 美元和 900 美元。SpyRL 在 creative-writing evaluation 中全面超过 Qwen3.5-27B-RaR；面对 GPT-4o-RaR 时整体接近，在 novelty 与 emotion 上更强。

这些结果支持了 RLSVR 的核心假设：开放任务可以通过 task transformation 获得有效的 verifiable learning signal。仍需留意训练规模。主要 GRPO 实验使用单节点 8 GPUs、每 prompt 8 rollouts、每 batch 128 prompts、100 iterations。每个 episode 又包含五个 performing outputs 和 detection interactions。省掉 external verifier 的同时，rollout 量、context length 与 multi-agent orchestration 都会增加。

## 我最关心的三个问题

第一，proxy game 的 Goodhart 风险。

模型长期训练后，可能找到让 detector 困惑的写法，这些写法未必等价于更好的 summary 或 story。需要持续监测 vote-quality correlation，并在未参与训练的新 detectors 和 human evaluation 上检查稳定性。

第二，什么任务适合构造 latent variable。

Summarization 很自然，因为可以遮掉内容。Creative writing 仍有较多设计空间。代码审查、research synthesis、planning 或 tool use 能否找到不泄露 shortcut、又与目标质量紧密相关的 information asymmetry，会决定方法的覆盖范围。

第三，multi-agent rollout 的 systems economics。

LLM judge 的成本容易计算，self-play 的 GPU 成本同样需要完整核算。Player 数、output length、detection context、stage switching、vLLM batching 与 learner utilization 共同决定每单位 improvement 的代价。对于 distributed RL 系统，这些参数本身就是算法的一部分。

## 为什么今天选它

RLSVR 给出了一种很有生命力的 post-training 抽象：不要直接逼近缺失的 reward，先寻找一个能够自动产生 ground truth 的环境变换。

这条路线把问题交给 task design、environment design 和 self-play dynamics。SpyRL 只是第一个具体实例。真正值得继续追的是，更多开放能力能否被改造成带有可验证 latent state 的交互任务，以及这些 proxy rewards 在规模化训练中能否长期保持与真实质量一致。

## 另外两篇

第二篇是 **To Add Is Machine, To Delete Is Human: Measuring and Mitigating Deletion Avoidance in LLM Code Editing**。

它发现 coding models 即使定位到正确文件，也经常回避删除。五个 SWE-bench Verified frontier models 对 developer patch 中 required deletions 的 recall 最高只有 71.7%；模型在超过 92% 的样本里能找到相关文件，却在不到 52% 的情况下删掉准确行。29% 的 passing patches 会保留目标代码，再套上 guard 或 fallback，作者称为 Guard-and-Go。

当测试明确要求旧代码消失时，四个 frontier models 的通过率从 63.2% 跌到 41.9%。论文还发布 200 个 deletion-only real tasks 组成的 CanItDelete benchmark，并初步表明 deletion-focused post-training 能缓解问题。它很适合拿来审视 SWE agent 的训练分布和 test adequacy。

第三篇是 **SAF-OPD: Stable Advantage Fusion for On-Policy Distillation**。

RLVR 给出 bounded response-level advantage，OPD 给出 dense token-level teacher advantage。固定系数融合时，OPD 的幅度可能远高于 RLVR，并在训练后期持续拉向 teacher，引发 entropy collapse，也限制 student 超过 teacher 的探索。

SAF 只处理 OPD advantage，依次做 sparsify、compress、warm-up 和 anneal。它在 Qwen3 1.7B、4B、8B 的 math 与 code training 上稳定超过固定系数融合，六个 model-domain settings 的 aggregate gain 为 0.51 到 2.70 percentage points。方法很小，诊断很清楚，值得在现有 GRPO + distillation pipeline 中快速验证。

论文：<https://arxiv.org/abs/2607.23802>

另外两篇：<https://arxiv.org/abs/2607.28887>、<https://arxiv.org/abs/2607.29209>
