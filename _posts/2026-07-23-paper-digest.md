---
title: "Paper Digest: 2026-07-23"
categories: [Paper Digest]
tags: [AI, DeepSeek-V4, Ascend, Post-Training, Distributed Training, Code Agents, RL]
---

今天最值得看的 paper，我会选 **SLAI T-Rex: Full-Parameter Post-training of the DeepSeek-V4 Family on Ascend SuperPOD**。

这是一份少见的全栈技术报告。它从 1.6T 参数 DeepSeek-V4-Pro 的 full-parameter post-training 出发，一路写到 Ascend SuperPOD 上的 parallelism、communication、optimizer offload、kernel optimization，再落到 Operations Research 场景的 CPT 与 SFT。

最终数字很醒目：Model FLOPs Utilization 从 11.67% 提高到 34.22%，相对开源 baseline 提升 2.93 倍。

更有价值的是数字背后的 profiling 过程。作者没有先认定某种并行策略更先进，再想办法把它搬到 Ascend 上。他们先拆解 step time：

`T_step = T_comp + T_non-overlapped + T_bubble + T_idle`

然后用实际 trace 判断哪一项真正主导端到端延迟。

在这套 1.6T MoE workload 中，expert-parallel all-to-all 只占 communication time 的 14.5%。Pipeline point-to-point transfer 占 36.7%，主要来自 receive-side stall、stage bubble 与 load imbalance；tensor/data-parallel AllReduce 占 48.4%。

这个分布直接改变了训练调度的选择。

DeepSeek 的 DualPipeV 擅长把 expert all-to-all 与计算重叠，但它需要每张卡同时保留两份 model chunks 和更多 in-flight activations。Ascend 910C 每张 NPU 只有 64GB HBM，而更节省显存的 VPP 已经占到 51.7 至 54.6GB。继续使用 DualPipeV 会迫使系统减小 micro-batch 或增加 activation recomputation。

更关键的是，DualPipeV 重点优化的 A2A 并非这里的主要瓶颈。作者最终选择 Virtual Pipeline Parallelism，用更细粒度的 virtual stages 压缩占比更大的 pipeline bubble。

这件事看起来朴素，却是整篇报告最值得带走的原则：distributed training 的策略优劣没有脱离 workload、topology 与 memory envelope 的统一答案。Trace 决定优化顺序。

第二个关键动作是 **expert-tensor parallel folding**。

为了让模型、activation 和 optimizer state 塞进显存，naive recipe 需要较高的 tensor parallel degree，例如 TP=4。但 dense attention matrix 被切得越碎，collective 越频繁，GEMM efficiency 也越差。

SLAI T-Rex 把 attention 与 MoE 的 parallelism 选择解耦。Attention 使用 TP=2，让 dense GEMM 保持更大 shape；MoE 层使用 ETP=1，保留完整 expert width，并把 expert-tensor parallel group 折叠进 expert parallel group。这样一次 all-to-all 就能完成 dispatch，额外的 ETP all-gather 与 reduce-scatter 被消除。

MoE dispatcher 里还有一个很具体的调度优化。

Token routing 完成后，系统需要先得到每个 expert 的动态 split size，随后才能发起 A2A。这会在 `before_ep_alltoall` 处产生可见的 synchronization stall。Shared expert 的计算只依赖 MoE layer input，不依赖 routing result，所以作者把 shared-expert forward 提前，用它覆盖这段同步等待。

这里没有引入新的数学结构，只是把时间线上原本闲置的一段填成有效计算。大型训练系统的效率往往就藏在这样的依赖关系里。

第三个动作是 **double-buffered swap optimizer**。

Full-parameter AdamW 需要 FP32 master weights、first moment 和 second moment。Trillion-parameter model 上，optimizer states 很快成为 HBM 的主导开销。把状态 offload 到 host memory 可以省显存，但传统 swap optimizer 会把 H2D、update、D2H 串行排在 critical path 上。

这篇报告把 swapped states 切成 chunks。Fused AdamW 更新当前 chunk 时，下一个 chunk 从 host prefetch，前一个更新完成的 chunk 异步写回。Swap-in、compute、swap-out 形成 pipeline。

它的收益不只是一张显存账单。省下来的 HBM 可以容纳更多 activation 和更大的 micro-batch，进而减少 recomputation 与 pipeline bubble。Memory optimization 最终又回到了 step time。

报告里还有一个很有意思的系统，叫 **AuraKernel**。

AscendC kernel optimization 的难点与 CUDA 不完全相同。一个完整 operator 同时包含 host-side scheduling 与 device-side kernel implementation，agent 需要理解 tiling、memory movement、launch configuration，以及 host-device contract。复杂 sparse-attention kernel 还包含长距离 code dependency、多级 memory hierarchy 和 stream coordination。

AuraKernel 用三层流程处理这个问题。

第一层是 solver-guided tiling。系统先判断 kernel 属于 compute-bound、memory-bound、scalar/control-bound 还是 communication-bound，再把 AI Core partition、GM/L2/L1/L0/UB tile、DMA overlap、buffer count 和 loop order写成约束优化问题。它优化的目标是 steady-state pipeline 中最慢的 stage：

`min max(T_MTE1, T_MTE2, T_MAC, T_FixPipe, T_scalar)`

Solver 给出的结果包含初始 tiling，也包含 bottleneck diagnosis。这样 agent 不必在无关的代码改动上消耗大量 iteration。

第二层是 hardware-grounded loop。每个候选都进入 compile、correctness gate、profile 的闭环。K-Search 保存可以 branch、rollback 的优化树，只有验证过的 improvement 才会形成 checkpoint。

第三层是 skill distillation。成功的 indicator、strategy 与 code pattern 会被沉淀成可复用 skill，供后续 kernel episode 检索。

这个设计把 kernel agent 从一次性的代码生成器，变成有约束初始化、真实硬件反馈、可回滚搜索与跨任务记忆的 optimizer。

论文的后半部分把基础设施用于 Operations Research specialization。作者使用 DeepSeek-V4-Flash 构建 CPT 与 SFT pipeline，数据来自领域资料与 solver-verified synthetic documents。SFT 数据只有 10K，覆盖四类任务与三种问题表示。最终模型的平均 zero-shot Pass@1 达到 71.81%，超过 GPT-5.4-Mini 3.98 points，也超过 base DeepSeek-V4-Flash 11.27 points。

前 15 页主要覆盖 infra，data pipeline 与完整实验在后半部分。今晚的深度分析值得继续追三个问题：

1. 11.67% 到 34.22% 的 MFU 增益，parallelism、kernel 和 fusion 各自贡献多少。
2. Double-buffered optimizer offload 在不同 micro-batch 与 recomputation 配置下的 wall-clock 曲线。
3. OR specialization 中 CPT、SFT cleaning、solver verification 和 progressive training 的独立增益。

今天第二篇是 **Beyond Fail-to-Pass: Iterative Hardening of Co-Generated Bug Reproduction Tests and Fixes**。

它指出 coding agent 里的一个隐蔽问题：generated test 和 generated patch 可能一起犯错。只要两者彼此一致，fail-to-pass check 依然可以通过。

作者把满足 fail-to-pass 的 bug reproduction test 继续分成 rigorous 与 lax。Lax test 能复现表面症状，却仍然接受 plausible-but-wrong patches。CoHarden 先生成 test，再用 surviving mutation patches 迭代强化 test 与 fix，直到当前 test 不再放过这些 regression。

在 SWE-bench Verified 上，CoHarden 达到 69.4% Resolved 和 78.9% fail-to-pass。Resolved 相对最强 fix-only baseline 提高 9.6 points，相对最强 co-generation baseline 提高 7.9 points。

这个结果对 agent training 也很重要。把 generated test 当 verifier 或 reward 之前，先要检查它是否能区分正确修复与一组看似合理的错误修复。Self-consistency 本身无法承担 correctness guarantee。

第三篇是 **Beyond Euclidean Clipping: Overcoming Exploration Collapse in LLM RL via Riemannian Isometric Policy Optimization**。

它把 PPO-Clip 的 exploration collapse 归因于 geometry mismatch。Euclidean clipping 会让 low-probability region 的更新过于保守，同时允许 high-probability region 更激进地移动。对于依赖罕见 reasoning branch 的任务，这会持续压缩有效探索。

RIPO 在 policy 的 Riemannian manifold 上构造 isometric update，试图同时控制探索与优化稳定性。论文在七个 competition-level benchmarks 上测试，报告 AIME 2024 相对 GRPO 最高 60% 的提升。

这个数字需要在 matched rollout、token budget 与 wall-clock 下验证。论文提出的问题却很尖锐：LLM RL 中的 clipping rule 可能正在用错误的距离度量塑造 policy。

今天三篇 paper 给出三个值得带回工程现场的信号：

1. 大规模 MoE training 的并行策略必须服从真实 trace、topology 与 HBM 约束。
2. Coding agent 生成的 verifier 需要接受 adversarial hardening，避免 test-fix error coupling。
3. RL exploration 的瓶颈可能藏在 policy update 所采用的 geometry 里。

如果只读一篇，先读 `2607.20145`。它把 DeepSeek-V4 full-parameter post-training、Ascend SuperPOD、agentic kernel optimization 与 CPT-SFT specialization 放进了一条完整链路。更难得的是，它给出了足够具体的系统取舍，让人可以逐项复现与质疑。

## 今日 3 篇精选

### 1) SLAI T-Rex: Full-Parameter Post-training of the DeepSeek-V4 Family on Ascend SuperPOD
- 链接: https://arxiv.org/abs/2607.20145
- 摘要速读: 在 Ascend CloudMatrix384 SuperPOD 上优化 1.6T DeepSeek-V4-Pro full-parameter training，通过 parallelism、communication overlap、optimizer offload、kernel agent 与 fusion，把 MFU 从 11.67% 提高到 34.22%。
- 为什么重要: 它用端到端 profiling 解释了为何该 workload 选择 VPP、ETP folding 和 double-buffered swap optimizer，并进一步完成 solver-grounded CPT-SFT specialization。

### 2) Beyond Fail-to-Pass: Iterative Hardening of Co-Generated Bug Reproduction Tests and Fixes
- 链接: https://arxiv.org/abs/2607.19843
- 摘要速读: 用 surviving mutation patches 迭代强化 generated tests 与 fixes，避免错误 test 与错误 patch 相互证明。
- 为什么重要: SWE-bench Verified 达到 69.4% Resolved，并把 verifier quality 从 fail-to-pass 推进到能排除 plausible-but-wrong patches。

### 3) Beyond Euclidean Clipping: Overcoming Exploration Collapse in LLM RL via Riemannian Isometric Policy Optimization
- 链接: https://arxiv.org/abs/2607.10169
- 摘要速读: 用 Riemannian isometric policy update 修正 PPO-Clip 在 low-probability region 过度保守的问题。
- 为什么重要: 它为 LLM RL 的 exploration collapse 提供了 geometry-level diagnosis，并在 AIME 2024 报告相对 GRPO 最高 60% 的提升。

## 一句话结论

今天最强的信号是：**大模型训练系统里，优化策略的名字没有 trace 重要。先找出真正占据 critical path 的那一段，再决定 parallelism、memory 和 kernel 应该如何协同。**
