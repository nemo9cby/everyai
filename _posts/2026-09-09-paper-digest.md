---
title: "Paper Digest: 2026-09-09"
categories: [Paper Digest]
tags: [AI, LLM, Post-Training, Reinforcement Learning, Code Agents, Distributed Systems]
---

今天最值得看的 paper，我会选 **Miles v0.1: Production-Level Post-Training**。

Miles 是一套面向 frontier post-training 的开源 full-stack system。它把 rollout、trainer、weight synchronization 和异步执行放进同一套架构，并用一个足够有分量的 case study 收尾：在 64 张 NVIDIA GB300 上，对 GLM-5.2 744B-A40B 做 fully asynchronous agentic RL，任务是 terminal-use coding，前 30 个 measured steps 的 median step time 为 263 秒。

这篇 technical report 的价值很直接。很多 RL 论文会把某个 objective、某个 sampler 或某个通信优化单独拿出来测。真正把系统跑起来时，性能和稳定性取决于这些部件之间的接口：rollout engine 何时看到新权重，trainer 如何切分模型，权重怎样跨集群拓扑传播，异步执行允许多大的 policy staleness，以及失败后如何定位问题。

## 一条完整的 post-training 数据路径

Miles 的 rollout engine 建在 SGLang 上，trainer 可以选择两套 backend：

- NVIDIA Megatron-LM，适合大模型和复杂并行策略。
- PyTorch FSDP，保留更贴近 PyTorch 生态的训练路径。

系统提供三种 weight-synchronization transport，用来适配不同部署拓扑。这个设计很重要，因为 rollout 与 training 往往由不同 GPU pool 承担。同步速度决定 rollout policy 的新鲜度，通信方式也会影响 GPU idle time、扩缩容难度和故障恢复。

同一架构支持 full-parameter RL、LoRA RL、on-policy distillation、SFT 和 true-on-policy rollout-training alignment，还扩展到了 diffusion model。这让团队可以在相同的 rollout、调度与监控框架中切换训练范式，减少系统差异对实验结果的干扰。

## 744B coding-agent RL 是真正的压力测试

论文展示了 GLM-5.2 744B-A40B 的 fully asynchronous agentic RL。Agent 在 terminal 环境里完成 coding tasks，意味着 rollout 不再是一段纯文本生成。每条 trajectory 会混合长上下文、工具调用、环境等待、失败重试和不同长度的响应。

这种 workload 会同时考验几件事：

1. Rollout engine 是否能在大量长短不一的 trajectory 之间维持较高 GPU utilization。
2. Trainer 与 rollout workers 是否能并行推进，并限制 policy staleness。
3. 744B MoE model 的权重更新能否及时传播到 serving side。
4. 环境延迟、模型生成和训练计算分别占据多少 step time。

64 张 GB300、median step time 263 秒给了一个真实的规模锚点。更值得从正文里追踪的是这 263 秒如何分解，以及吞吐、稳定性和 on-policy fidelity 之间的取舍。

## 对分布式 RL 团队的意义

Miles 把四个常被分开讨论的目标并列为 first-class goals：accuracy、efficiency、reliability 和 scalability。生产系统很难只优化其中一个。更激进的异步执行可以提高硬件利用率，也可能让 rollout 落后于当前 policy；更频繁的权重同步能降低 staleness，却会增加网络压力和停顿。

这篇报告适合作为系统 design review 的参照物。读的时候可以带着几个具体问题：

- True-on-policy 在系统里由哪些同步边界保证？
- Fully asynchronous 模式如何定义并测量 policy staleness？
- 三种 weight transport 分别适合哪些网络和部署拓扑？
- SGLang rollout 的 bottleneck 是 decoding、KV cache、环境 I/O，还是调度？
- Megatron 与 FSDP backend 在可扩展性、可定制性和调试成本上如何取舍？

## 另外两篇

第二篇是 **NeoHorse-1: Towards Recursive Self-Improvement via Agentic Post-Training with Routing Harness**。

NeoHorse 把 model routing 产生的在线记录转成训练信号。系统保存 capability demand、service tier、reasoning、tool calls 和 harness context，再经过结构检查、多维语义评估与 subscene labeling，进入三阶段 SFT curriculum 和 routing-guided on-policy distillation。4B 模型在 11 个 agent、tool-use、coding 与 instruction-following benchmark 上的 macro-average 从 58.94 提升到 64.87。

第三篇是 **Online Draft Co-Training for Speculative Decoding in Large-Scale, Long-Context RL Post-Training**。

它让 speculative draft model 在线追踪持续更新的 RL policy，并解决大规模 co-training 的两个系统问题：context parallelism 下的 branch attention，以及 pipeline parallelism 下跨 stage 的 target feature transport。实验扩展到 122B model 和 256K context，目标是直接降低 rollout 与端到端 RL step time。

论文：<https://arxiv.org/abs/2609.08368>

另外两篇：<https://arxiv.org/abs/2609.08183>、<https://arxiv.org/abs/2609.07108>
