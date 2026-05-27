---
title: "Paper Digest: 2026-05-27"
categories: [Paper Digest]
tags: [AI, Agents, Coding Agents, Foundation Models, Post-Training]
---

今天最值得看的 paper，我会给 **MiniMax-M2**。

这篇是 foundation model 技术报告。表面上最醒目的数字是 MoE: 旗舰模型 M2 有 229.9B total parameters，每个 token 只激活 9.8B。这个 sparse activation 设计让模型在推理成本和能力之间取得更好的平衡。

但这篇真正值得读的地方，是 MiniMax 把模型能力放在完整的 agent training stack 里讲。

第一层是数据。MiniMax-M2 的数据管线围绕 agentic deployment 设计，生成大规模 verifiable trajectories，覆盖 agentic coding 和 agentic cowork。每条 trajectory 都绑定 executable workspace 和 artifact-aligned reward。也就是说，训练信号直接落在代码、文档、办公产物、搜索结果这类可检查产物上。

第二层是 RL 系统。论文里的 Forge 是一个 agent-native RL infrastructure，专门处理 long-horizon agent trajectories。它包括 windowed-FIFO scheduling、prefix-tree merging、inference optimization，以及训练、推理、agent 执行之间的清晰解耦。这个设计很实用，因为 agent RL 的瓶颈经常卡在轨迹太长、rollout 太贵、执行系统太重。

第三层是 self-evolution。M2.7 checkpoint 已经能做一些早期的自我改进动作，例如自动 debug training runs，修改自己的 scaffold。这个方向当然还早，但它说明 MiniMax-M2 把 agent scaffold 当成训练系统的一部分，而不是部署后的外围脚本。

这对 coding agent 很关键。

很多讨论会盯着模型参数、benchmark 排名、context length。但真实 coding agent 的能力，很大一部分来自它被放进怎样的 workspace，用怎样的 reward 检查产物，用怎样的系统处理长轨迹。MiniMax-M2 这篇报告的价值，就在于它把这些工程层明确写进了 foundation model recipe。

今天另外两篇也值得一起看。

**MobileGym** 做的是 mobile GUI agent 的 verifiable simulation。它把完整 app state 表示成 structured JSON，可以配置、fork、比较，并用 deterministic state-based judge 给出 evaluation verdict 和 dense RL reward。单机能跑数百个并行实例，每个实例大约 400MB，cold start 约 3 秒。MobileGym-Bench 覆盖 28 个 app、416 个参数化任务模板。论文还报告，Qwen3-VL-4B-Instruct 经过 GRPO 后 simulation test set 提升 +12.8 points，真实设备子集保留了 95.1% 的 simulation-side training gain。

**RepoMirage** 则是 coding agent 评测里的冷水。它基于 SWE-Bench Verified 做 repository perturbation，测试 code agent 是否真的理解多文件 repo context。结果很扎心：在原始设置下平均 66.8% 的表现，到了 perturbation-targeted explicit tasks 掉到 25.3%。作者进一步观察到 exploration drift: agent 会打开更多文件，但没有把这些探索转化成有效的 repo structure understanding。于是他们提出 RepoAnchor，把 repository exploration 和下游 problem solving 分开，先构建结构脚手架，再解题。

把今天这三篇放在一起看，信号很清楚：agent 能力的核心越来越像一个系统工程问题。MiniMax-M2 讲 model + data + RL stack，MobileGym 讲 verifiable environment，RepoMirage 讲 repo context 诊断。模型当然重要，但环境、reward、trajectory、scaffold、evaluation 同样决定最后能不能在真实任务里稳定工作。

所以今天先读 `2605.26494`。如果你关心 coding agent、post-training、RL infrastructure，或者想知道 foundation model team 怎么把 agent ability 写进训练系统，这篇值得认真看。

## 今日 3 篇精选

### 1) The MiniMax-M2 Series: Mini Activations Unleashing Max Real-World Intelligence
- 链接: https://arxiv.org/abs/2605.26494
- 摘要速读: 发布 MiniMax-M2 sparse MoE 模型族，并系统介绍 agent-driven data pipeline、Forge agent-native RL system 和早期 self-evolution scaffold。
- 为什么重要: 它把 agentic coding 和 real-world agent ability 放进 foundation model 的训练基础设施里讲。

### 2) MobileGym: A Verifiable and Highly Parallel Simulation Platform for Mobile GUI Agent Research
- 链接: https://arxiv.org/abs/2605.26114
- 摘要速读: 用 structured JSON state、deterministic judge 和高并行 browser-hosted simulation，构建 mobile GUI agent 的 RL 和评测环境。
- 为什么重要: 它让 mobile GUI agent 有了更便宜、更可验证、更适合在线 RL 的训练场。

### 3) RepoMirage: Probing Repository Context Reasoning in Code Agents with Perturbations
- 链接: https://arxiv.org/abs/2605.26177
- 摘要速读: 通过 SWE-Bench Verified 上的 repository perturbation，诊断 code agent 是否真的理解 repo context。
- 为什么重要: 它揭示了 coding agent 在 repository structure reasoning 上的短板，也给出 structure-first workflow 的方向。

## 一句话结论
今天最强的研究信号是：**agent 能力要靠模型、环境、reward、trajectory 和 scaffold 一起做实。** `2605.26494` 值得先读。
