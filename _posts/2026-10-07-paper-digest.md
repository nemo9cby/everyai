---
title: "Paper Digest: 2026-10-07"
categories: [Paper Digest]
tags: [AI, LLM, On-Policy Distillation, Post-training, Quantization, Coding Agents]
---

今天最值得看的 paper 是 **Rethinking Cross-Tokenizer On-Policy Distillation: From Alignment Coverage to Supervision Reliability**。

它研究一个越来越常见的 post-training 场景：teacher 和 student 使用不同 tokenizer 时，怎样做 on-policy distillation？

直觉上，token 对不齐就应该设计更复杂的 span alignment，把更多位置纳入监督。论文给出的实验结果很克制：严格的 1:1 对齐已经覆盖了大部分 student-generated tokens。把近似匹配的 span 也塞进 loss，监督覆盖率虽然达到 100%，模型准确率反而下降。

## 跨 tokenizer 蒸馏的两层对齐

On-Policy Distillation 让 student 在自己的生成轨迹上接受 teacher 的分布监督。Teacher 和 student tokenizer 不同时，需要解决两层问题：

- sequence alignment：两个 tokenizer 切出的 token 边界能否对应
- vocabulary alignment：对应位置上，两边能够比较哪些 token probability

论文测试了三个异构 teacher-student pair，任务覆盖数学推理和代码生成。尽管 vocabularies 差异明显，严格 1:1 alignment groups 仍然覆盖了多数 student tokens。

作者进一步检查这些严格对齐位置上的 probability mass。结果显示，共享 vocabulary 平均保留了 teacher 和 student 几乎全部的概率质量。换句话说，tokenizer 不同看起来制造了很大的 vocabulary gap，真正落到 student 自己生成的轨迹上，可用的监督比预想中完整得多。

## Top-16 reverse KL 已经足够

有了 strict positions，最直接的方法是在全部 shared vocabulary 上计算 reverse KL。论文发现，还可以进一步压缩监督：每个位置只保留 student 选出的 shared-vocabulary top 16 tokens。

这个 compact loss 的准确率与 full shared-vocabulary OPD 相当，并超过论文评估的 cross-tokenizer baselines。

这对训练系统很有现实意义。跨模型蒸馏通常需要传输或保存 teacher logits。若每个 strict position 只需要一个很小的候选集合，通信量、存储量和 KL 计算都会下降。Student 负责选择局部 support，也让监督集中在当前 policy 真正可能采取的 token 上。

## 100% supervision coverage 为什么更差

为了覆盖 tokenizer mismatch groups，作者加入 span log-probability 的 MSE supervision。它让每个位置都获得训练信号，却降低了最终准确率。

论文没有停在 outcome comparison。作者取只使用 strict loss 训练得到的 checkpoints，分别计算 strict objective 与 span objective 的 gradients：

- 两组 gradient 的方向一致性很弱，有时为负
- span gradient 相对 strict gradient 的规模随训练增长
- 较大的 mismatch-span update 会逐渐盖过更可靠的 strict signal

这是一种很实用的 post-training 诊断方式。新 loss 带来更多 labelled positions，并不等于带来更多有效学习。先看 gradient cosine、relative norm 和 downstream accuracy，才能判断辅助目标是在补充监督，还是在持续拉偏主目标。

## 对 post-training pipeline 的启发

一套务实的 cross-tokenizer OPD pipeline 可以从最小方案开始：

1. 找出严格 sequence alignments
2. 测量 shared vocabulary 保留的 probability mass
3. 用 student-selected top-k support 计算 reverse KL
4. 监控不同 supervision channels 的 gradient cosine 与 norm ratio
5. 只有在下游指标稳定改善时，才加入 span-level approximate alignment

这里真正值得带走的原则是 supervision reliability。训练目标的覆盖率只是一个描述统计，更新方向是否可信才决定模型最终学到什么。

## 今天另外两篇相关论文

**TRACE** 处理大规模 MoE RL 中的 FP4 rollout。它让 rollout-side quantization outcome 指导 training-side rounding，直接缩小两个 execution path 的数值差异。论文报告 joint FP4 weight/activation 与 FP4 KV-cache rollout 可以维持接近 BF16 的 RL 表现，同时把 rollout 加速到最高 5.4x。对于 rollout 占据大部分集群时间的系统，这是一条很硬的工程线索。

**HERMES** 把 repository artifacts 封装成带 resident LLM 的 executable Dev-Primitives，再通过 dependency-aware activation 和 bug diagnosis 选择需要参与的组件。它在四个 software-engineering benchmarks 上平均超过 matched harnesses 12.4%，并展示了小模型 component agents 配合强 routing 后的成本优势。最值得关注的是它对 context ownership 的处理：程序知识留在组件附近，runtime evidence 决定哪些局部状态需要被激活和修改。

论文：<https://arxiv.org/abs/2610.08448>

另外两篇：<https://arxiv.org/abs/2610.07767>、<https://arxiv.org/abs/2610.07832>
