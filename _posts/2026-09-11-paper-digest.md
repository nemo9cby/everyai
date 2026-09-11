---
title: "Paper Digest: 2026-09-11"
categories: [Paper Digest]
tags: [AI, LLM, Foundation Models, Pretraining, Latent Space, Post-Training]
---

今天最值得看的 paper，我会选 **NCP-ArchPreview Technical Report: Moving towards Latent Space Language Models through Next Concept Prediction**。

语言模型一直在做一件极其局部的事：根据前面的 token，预测下一个 token。这个目标简单、稳定、容易扩展，也把模型牢牢绑定在词表和表面语言上。

NCP-ArchPreview 给模型增加了第二个预测目标。它把连续多个 token 的 hidden states 压缩成离散 concept，同时学习下一个 token 和下一个 concept。Concept 的预测结果随后重新注入 token decoder，继续参与标准 autoregressive generation。

这篇 technical report 最有分量的地方是规模。作者把架构训练到 8.9B 参数和 5.73T tokens，并公开 weights、training recipe、inference scripts 和中间 checkpoints。它提供了少见的 trillion-token 证据，说明 latent-space prediction 可以进入真正的 foundation model pretraining。

## 模型如何同时看 token 和 concept

NCP-ArchPreview 基于 OLMo-3-7B，把原来的 32 层 Transformer 拆成三个部分：

1. 16 层 Token Encoder
2. 8 层 Concept Module
3. 16 层 Token Decoder

Token Encoder 先生成 token-level hidden states。模型每四个 token 做一次 mean pooling，得到长度缩短为四分之一的 concept sequence。

连续向量还不够。作者使用 product quantization 建立离散 concept vocabulary：一条 concept vector 被拆成 32 段，每段对应一个包含 128 个 entries 的 codebook。组合空间因此可以很大，同时避免维护一个巨型单体词表。

Concept Module 根据前面的 concepts 预测下一个 concept。预测结果恢复到 token resolution，并通过保持因果性的 shift 注入 Token Decoder。最终 decoder 仍然输出普通 token，现有的 autoregressive inference interface 可以保留。

训练目标由三部分组成：标准 Next Token Prediction、用于学习 codebooks 的 VQ objective，以及 Next Concept Prediction。Token Encoder、Concept Module 和 Token Decoder 端到端联合更新。

这套设计有一个很实际的 compute 特征。Concept Module 的 sequence length 只有 token stream 的约四分之一。一个 concept block 拥有接近普通 Transformer block 的参数量，分析 FLOPs 却低很多。模型可以在较短的 latent sequence 上增加表示容量。

## 5.73T tokens 上的结果

作者沿用 OLMo-3 的两阶段数据 curriculum。Stage 1 使用 Dolma 3 Mix，Stage 2 使用 Dolma 3 Dolmino，并沿用 OLMo-Core 的 30 个 benchmark families 做评估。

Stage 1 中，NCP-ArchPreview 只消耗 OLMo-3-7B 51.3% 的训练 tokens，就达到后者的最终 loss，相当于按 token 计算 1.95 倍的 convergence speedup。完成训练后，它的 downstream macro-average 高 2.45 分，GSM8K 高 5.99 分。提升主要集中在 math、code 和 non-STEM multiple-choice tasks，其中 MATH 的相对提升达到 18%。

Stage 2 的优势缩小。它用 66.2% 的 tokens 达到 OLMo-3-7B 的最终 loss，最终 loss 低 0.027，下游 macro-average 高 0.59 分。

这里还有一个很值得训练团队注意的结果。Stage 2 的几个配置中，training loss 持续降低，下游表现却持续变差。作者认为原因可能是 training mixture 和 downstream distribution 不匹配。Stage 2 只有大约 10% code data，aggregate loss 变好时，低占比 domain 的能力依然可能受损。

Loss curve 不能代替 capability evaluation。数据 mixture 改变之后，团队仍然需要 domain-aware proxy benchmarks 和 checkpoint selection。

## 提升来自架构，还是更多参数

NCP-ArchPreview 有 8.94B 参数，比 7B baseline 更大。论文因此做了 parameter-aligned 和 computation-aligned comparison。

作者构造了三个 OLMo-3 baselines：标准 32 层模型、34 层 compute-aligned 模型，以及 40 层 size-aligned 模型。NCP-ArchPreview 的八个 concept blocks 按参数量约等于八个普通 blocks，按计算量只相当于约两个 blocks，所以整体近似 40-block parameters 和 34-block compute。

结果显示，它明显优于标准模型和 compute-aligned baseline。在只使用 size-aligned baseline 85% 计算量的情况下，它接近后者的 loss。

Progressive ablation 也给出一致方向。加入 Concept Module 后 loss 下降，再加入 hierarchical residual routing 后继续下降，最后加入 NCP objective 又获得额外改善。Scaling-law experiment 在多个 FLOPs budgets 上重新搜索 model size、data allocation 和 hyperparameters，报告相对 OLMo-3 的 1.74 倍 compute efficiency。

这些实验让论文的核心 claim 更可信。优势同时来自较短 latent sequence 提供的额外容量、跨模块信息流，以及 concept-level supervision。

## Concept space 还可以做什么

学到的 concept vocabulary 也成为一个很轻的 adaptation interface。

论文在 code、math 和 knowledge domains 上继续训练时，只更新 17M 参数的 VQ module。对需要快速 domain adaptation 的场景，这提供了一个值得验证的路径：保留主模型 weights，调整模型如何离散化和组织 concept space。

作者还把 concept representations 注入 DFlash2 speculative drafter，mean accepted length 提高 4.17%，额外开销很小。Concept Module 在这里扮演了语义级 future signal，帮助小 drafter 提出更容易被主模型接受的 tokens。

同一个 latent interface 同时连接 pretraining efficiency、domain adaptation 和 speculative decoding，这是 NCP-ArchPreview 最有想象力的部分。

## 对训练基础设施的启发

这类模型会给 distributed training stack 增加新的观测对象：

- NTP、VQ、NCP 三条 loss 的独立曲线
- 每个 product-quantization codebook 的利用率与 collapse 风险
- Token Encoder、Concept Module、Token Decoder 的 gradient 和 optimizer telemetry
- matched-parameter、matched-FLOPs baselines
- 按 domain 划分的 proxy evaluation
- concept adaptation 与 full-model tuning 的成本和迁移效果

论文还提到 matrix optimizer 下的 attention-logit stabilization。新的模块和目标进入 trillion-token training 后，稳定性机制会和 architecture gain 一样重要。只看最终 benchmark，很容易把优化器、数据配比和 latent objective 的贡献混在一起。

NCP-ArchPreview 仍然留下不少问题。Concept chunk 固定为四个 token，mean pooling 也很简单；离散 concepts 是否形成稳定、可解释、跨语言的语义单元，还需要更多分析。Stage 2 的 code regression 说明更低 loss 没有自动解决数据分布问题。8.9B 上的结果也需要在更大规模和不同模型家族中复现。

不过，它已经给出了足够完整的第一步：新的 prediction target 可以在 5.73T-token 训练中稳定工作，带来 matched-compute 优势，并且保留普通语言模型的生成接口。

## 另外两篇

第二篇是 **Ecdysis: Efficient and Effective Training of Runtime Harnesses for LLM Agents**。

它针对 harness evolution 的浪费和过拟合问题。很多方法看到一次 agent failure 就修改 prompt、tool 或 runtime logic，容易为单个模型或单个任务打补丁。Ecdysis 先聚合多个任务里的 recurring failures，再区分 model-specific deficiency 和 systematic harness deficiency，通过多角色 diagnosis 生成修改方案。论文报告 harness training 最多加速 1.84 倍，最终 reasoning accuracy 提高 18.56%。

第三篇是 **Negative Self-Distillation: Learning to Reason by Avoiding Flaws**。

它质疑用 privileged information 生成过度自信的 reasoning trace，再让学生模仿的 OPSD 路线。NSD 让模型生成 question-specific negative condition，例如 careless reasoner，然后训练学生远离这条 flawed distribution。Dynamic gate 只选择 reasoning-critical tokens，降低对基础语言能力的伤害。这个思路很适合与 OPSD、RLVR 放进同一套 rollout pipeline 做 controlled comparison。

论文：<https://arxiv.org/abs/2609.10715>

另外两篇：<https://arxiv.org/abs/2609.11677>、<https://arxiv.org/abs/2609.11699>
