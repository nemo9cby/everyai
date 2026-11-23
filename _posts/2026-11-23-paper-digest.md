---
title: "Paper Digest: 2026-11-23"
categories: [Paper Digest]
tags: [AI, Coding Agents, GPU Optimization, EDA, Verification]
---

今天最值得看的 paper 是 **EDAGoat: An EDA-Aware Agent for Project-Level GPU Optimization**。

让 coding agent 优化 GPU code，评价标准很容易被简化成“benchmark 跑快了多少”。在真实工程里，一个候选程序还需要通过 correctness、hidden inputs、data race 和 domain-specific quality checks。EDAGoat 把这些约束放进 agent workflow，目标是交付一个经过完整验证的更快程序。

它选择了一个很硬的场景：优化专家已经反复调过的 GPU EDA solvers。任务覆盖 logic synthesis、placement、global routing 和 static timing analysis。Agent 需要同时理解 host code、GPU kernels、数据布局与电路设计约束，最后比较 complete-solve runtime。

## 两层 domain-aware workflow

EDAGoat 提供两种可以叠加的机制。

第一层是 optimization enhancement（OE）。研究者从多个 GPU EDA 项目中整理四类跨项目优化 pattern，每个 pattern 都包含：

- 适用的 optimization opportunity
- 具体 transformation
- 数据与执行依赖
- 使用前提和 fallback
- 对应的 validation template

这些 pattern 只提供方向。Agent 仍然要定位真实项目中的瓶颈，设计 thread mapping、memory mapping 与 synchronization，并完成代码修改。

第二层是 mandatory validation（MV）。Workflow controller 会要求 agent 提交当前 build 的 targeted-check evidence。缺少验证记录、检查失败或结果来自旧 build 时，候选程序无法进入性能评估和最终 nomination。

这个设计抓住了 agent optimization 中经常被忽略的问题：validation guidance 放在 prompt 里，并不保证 agent 真会在复杂修改后运行对应检查。把验证变成状态机中的硬门槛，行为才会稳定下来。

## Benchmark 只计算 qualified delivery

论文构建了五个 GPU EDA 项目、四类任务和 55 个 public/hidden circuit inputs。每个候选程序都要满足项目自己的 qualification contract，例如：

- Logic rewrite 要保持 Boolean equivalence，并满足 AND count 与 depth bounds。
- Placement 要保留 fixed cells，生成合法布局，并满足 wirelength 限制。
- Global routing 要覆盖全部 nets，保持 3D tree connectivity，并通过 wirelength 与 overflow 检查。
- 性能使用 complete-solve time，避免只优化一个 kernel 后隐藏整体回归。

一次 search 只有在所有公开与隐藏输入上通过 correctness 和 quality-of-result checks，才会进入 speedup 统计。每个模型、每个项目独立运行三次，因此 qualified delivery rate 与 speedup 同样重要。

## Pattern guidance 与强制验证解决不同问题

在 DeepSeek 上，generic prompting 的 15 次 search 只有 7 次合格。加入 OE 后，agent 在四个项目上找到更高的 best speedup，合格次数却降到 6/15。Domain knowledge 扩大了优化幅度，也增加了更激进修改带来的失败风险。

加入 MV 后，合格次数上升到 13/15。按项目计算的 mean speedup 几何平均值，从 generic prompting 的 1.11x、OE 的 1.36x，提高到 1.53x。MV 让 agent 在性能探索之外持续处理 correctness 与 QoR obligations。

在 GPT-6 Astra 上，EDAGoat 的 15 次 search 全部合格，并在五个项目上同时提高 mean 与 best complete-solve speedup。几何平均的 mean speedup 从 1.26x 提高到 1.57x，logic synthesis 的最佳结果达到 3.906x。

作者对失败进行归因后发现，generic agents 的主要问题包括 GPU write races，以及修改后遗漏原本必须执行的计算。它们都很适合用 edit-aware targeted checks 提前暴露。

## 对 LLM systems 的启发

这套方法可以直接迁移到训练和推理系统的 kernel optimization agent。一个可用的 workflow 可以把以下信息编码成 frozen skills：

- 优化 pattern 与适用的 tensor shape、dtype、memory layout
- numerical parity、race detection 和 determinism checks
- 对 convergence 敏感的算子误差边界
- 多硬件、多 batch size、多 sequence length 的 hidden evaluation matrix
- kernel speed 与 end-to-end throughput 的联合验收

Agent 每次修改 CUDA、Triton 或 NPU operator 后，需要用当前 build 产生验证证据，再进入性能比较。最终 nomination 还要通过独立 evaluator。这样可以防止 agent 利用 benchmark 缺口，也能把“偶然跑快一次”升级为可合并、可部署的工程结果。

论文的证据范围仍然有限。Benchmark 只有五个 EDA projects，optimization patterns 由研究者离线提炼，完整验证有时需要数小时。下一步值得测试自动 skill mining、验证预算分配，以及这套 qualified-delivery loop 在 LLM kernels、distributed collectives 和 serving runtimes 上的迁移能力。

## 今天另外两篇相关论文

**Clover-35B-A3B-Preview Technical Report** 介绍一个 35B total、3B active 的 MoE diffusion language model。它把 recurrent memory 与 block 内 bidirectional attention 结合，报告 SWE-bench Verified 73.6、Terminal-Bench 2.1 51.7，并在单张 NVIDIA GB300、BF16、batch size 1 下达到 680 tokens/s。Preview 中包含 internal workloads，后续需要验证真实 serving 场景下的 latency、memory 与 verifier cost。

**Git History Is a Pangenome** 测量了把完整 git history 用作 code-model training corpus 时的重复与泄漏。典型 repository history 相当于保留 19 份当前代码，成熟项目达到 89 份；exact-match dedup 几乎无效。跨版本 near-duplicate grouping 能把重复降到约两份，但需要 newest-first 才能避免一半文件保留旧版本。SWE-bench 的 merged fix 还会在中位数 18 个后续版本中继续存在，因此 benchmark decontamination 必须扫描全部版本。

论文：<https://arxiv.org/abs/2611.27196>

另外两篇：<https://arxiv.org/abs/2611.24636>、<https://arxiv.org/abs/2611.27154>
