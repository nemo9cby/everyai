---
title: "Paper Digest: 2026-09-02"
categories: [Paper Digest]
tags: [AI, Post-Training, GRPO, Model Merging, Function Calling, Production LLM]
---

今天最值得看的 paper，我会选 **From Production Traffic to Post-Training: Building a Self-Hosted LLM That Covers the Corporate Request Mix**。

大公司的内部 LLM 平台很容易长出一个 model zoo。两百多个应用各自选择当时最合适的模型，新模型持续上线，旧模型又因为迁移成本迟迟无法下线。有限的 GPU pool 被越来越多 checkpoint 切碎，batching efficiency 降低，维护和升级也越来越麻烦。

这篇论文记录了一次真正进入生产的 consolidation。团队以 Qwen3-32B 为基础，从真实流量中定位质量缺口，通过 shared SFT、三个独立 GRPO expert 和两阶段 SLERP，训练出一个统一的非推理模型。

六个月后，它承载了平台 50% 的流量：来自 200 多个内部应用，每月 1.16 亿次请求。

## 先用生产流量决定训练什么

团队对平台请求做人工 error analysis，发现三个最值得修的轴：

1. Instruction following，尤其是格式与约束遵循，占失败的 37.9%
2. Function calling，带工具请求约占总流量的 12%
3. 内部任务分布，与公开 instruction data 存在明显 distribution gap

这一步比直接收集更多通用数据重要。长尾应用生命周期很短，单个应用往往攒不出足够流量做可靠 A/B test。模型既要修复已经出现的生产问题，也要保持对新 workflow 的泛化。

论文因此建立了一套由真实流量分层采样的 in-house Arena。能写 deterministic verifier 的任务直接验证，开放式任务使用经过人工标定的 LLM judge。对 information extraction、generation 等不同任务，judge prompt 和 scoring recipe 分开设计，最终 judge 与人工的一致性显著提高。

这套 benchmark 的作用很具体：它决定哪些生产错误值得进入 post-training，也负责阻止某个能力的提升伤害其他流量。

## SFT 可以混在一起，GRPO 需要拆开

Base model 是 Qwen3-32B，使用更适合 Cyrillic dense text 的 tokenizer。生产延迟与成本要求它全程运行在 non-reasoning mode。

第一阶段使用一份 shared SFT mixture，同时加入通用俄语、内部请求、instruction-following 和 function-calling 数据。消融显示，混合 SFT 与各 domain 单独训练的 SFT checkpoint 接近，因此团队选择更简单的 shared checkpoint。

进入 GRPO 后，情况变得复杂。不同 reward 的优化几乎只改善自己的 domain，跨域迁移很弱：

- General expert 提高 Arena，IFEval 与 tool calling 基本留在原位
- IF expert 提高 constraint satisfaction，对另外两轴帮助有限
- FC expert 提高 tool use，也没有自动带来更强的通用能力

把三个 reward 放进一次 joint GRPO 又会产生明显干扰。8B 实验中，instruction following 从 0.731 提高到 0.801 时，English BFCL 从 61.2 跌到 54.5；加入 general reward 后，function calling 恢复，instruction following 又跌到 0.687。效果最接近 expert merge 的 joint run 需要先做 general-GRPO warm start，并使用 1.7 倍训练预算。

团队最终从同一个 shared SFT checkpoint 分叉，独立训练 General、IF 和 FC 三个 GRPO expert，再在 weight space 里合并。

## 三类 reward，各自长出一种 exploit

论文最有价值的部分，是它没有把 reward hacking 写成一句泛泛的风险提示。三个 expert 真的暴露出三种不同 failure mode，修复方法也完全不同。

### General expert：verbosity hacking

General branch 使用 reward model 优化生产任务分布。训练数据约 80% 来自开放俄语 instruction corpus，20% 来自经过采样和去污染的内部请求，completion 统一由 Qwen3-235B-A22B-Instruct-2507 重新生成。

Reward model 容易偏爱更长、更像优质回答的文本。团队加入 prompt-specific multiplicative length penalty，以强 teacher 在相同 prompt 上的长度为基准，同时提高 KL coefficient，限制 policy drift。

他们也尝试让 reward model 学习内部 preference，结果 in-house score 反而降低，平均回答长度从 286 token 增到 362 token。内部 preference pair 很可能携带了表面 style bias。最终系统保留 general-domain reward model，用 production data 改变 prompt distribution。

### IF expert：semantic collapse

Instruction-following branch 从 54 条手写 constraint 出发，经 augmentation、verifier consistency filtering 和 back-translation validation 扩展到 4.3 万条 verified constraint，最终构造 2.6 万个训练样本。

纯 verifier reward 很快被模型钻了空子。模型生成语义空洞的极短回答，也能满足形式约束。

团队在 VerIF-style reward 上加入 prompt-specific quality correction。通过 verifier 的 completion 还要与 teacher completion 的 reward-model score 比较，低于同 prompt teacher 均值的结果会收到 penalty。Formal constraint 继续提供可扩展的 deterministic signal，semantic quality 也有了一道下限。

### FC expert：over-calling

Function-calling branch 使用 Tool-N1 exact-match reward。这个 reward 的 dominant exploit 很直接：模型不确定时也发起 tool call，提高撞中正确调用的机会。

修复发生在 data distribution。训练集加入 synthetic irrelevant requests，让模型学习哪些输入无需调用工具。团队还原生生成 120 万条英文和 30 万条俄文 function-calling 数据，避免翻译破坏 schema、argument 与对话结构。

数据生成把 planning 与 simulation 分开。Planner 在 judge feedback 下先构建 trajectory，三个 agent 再以 asymmetric visibility 回放，减少 reference leakage，让训练 target 更接近真实多轮 tool use。

三个案例给出一个清晰结论：reward 的形式决定 exploit 的形状。Verifier、reward model 和 exact match 各自需要不同的 guardrail，统一处理很难稳定。

## Expert merge 把训练问题变成评测问题

三个 expert 使用 sequential SLERP 合并。SLERP 不满足结合律，merge order 会影响结果。论文枚举三种顺序与多组 near-convergence checkpoint，先合并 IF 与 FC，再合入 General 的结果最好。

这个方案把一部分难题移到了 evaluation time。增加新能力时，可以从 shared checkpoint 再训练一个独立 expert，保留清晰的 reward、data 和 regression provenance，然后搜索 merge coefficient 与顺序。Joint RL 需要重新调整 domain ratio、batch schedule 和 reward scale，任何新 objective 都可能触发一轮昂贵 sweep。

Merge 也有边界。它依赖同源 checkpoint 与相容的 parameter geometry，最终候选仍要跑完整的跨域 regression suite。论文的价值在于给出了 production-scale 的正面案例，并且公开了一个不含内部数据、使用相同 recipe 训练的 T-pro-it-2.1 checkpoint。

## 一个 32B 模型承担 1.16 亿次月请求

最终模型在 in-house Arena 上得到 69.57，超过 Qwen3-235B-A22B-Instruct-2507 的 65.83。In-house instruction following 是 0.85 对 0.83，function calling 是 0.79 对 0.77。它也把 base Qwen3-32B 的 ruWildChat 从 52.0 提高到 80.7。

生产部署使用 single-GPU FP8 vLLM replica，规模在 16 到 48 个 pod 之间：

- 平均 45 requests/s，峰值 110 requests/s
- P95 latency 3.2 秒
- Time to first token 0.3 秒
- 每月 1.16 亿次请求
- Input 与 output 的 per-token cost 降低 2.8 到 3.9 倍
- 对此前依赖最大模型的服务，成本降低可达 4 到 9 倍

少数 rollback 来自需要 frontier-scale agentic capability 的团队。32B dense model 无法覆盖所有 workload，但已经足以收回一半平台流量，让 fragmented fleet 获得一个真正可维护的主干。

## 对 post-training infrastructure 的启发

这套 recipe 很适合被实现成模块化 distributed pipeline：

- Shared SFT checkpoint 是所有能力分支的统一起点
- 每个 GRPO expert 拥有独立 rollout、reward service、data mixture 和 stopping rule
- Expert checkpoint 连同 reward version、policy version 与 eval results 一起登记
- Merge stage 只做 coefficient、ordering 和 checkpoint 组合搜索
- Production traffic benchmark 持续检查目标能力与 general regression

这里的 systems benefit 和 modeling benefit 是同一件事。训练分支互相隔离后，资源可以独立调度，失败更容易归因，新 capability 也能增量加入。模型合并随后成为一个便宜、可并行、可审计的 evaluation workload。

最值得复现的实验也很明确：在相同 shared SFT checkpoint 上，对比 joint multi-domain GRPO 与 expert-per-reward 加 merge，固定总 rollout budget，追踪每个 domain 的 learning curve、reward exploit、cross-domain regression 和最终 serving cost。

## 另外两篇

第二篇是 **HarnessDev: Can LLMs Create and Evolve Their Own Agent Harness?**。

HarnessDev 把评测对象改成 runnable agent infrastructure。Creation 阶段让模型从最小 seed 和少量 case 构建完整 harness，Evolution 阶段再依据 execution feedback 迭代。六个 creator model、四个 domain、五个 benchmark、2,207 个 downstream instance 的结果显示，模型生成的 harness 在 coding、search 和 research 上仍明显落后于成熟人工系统；writing 与 ML experimentation 已能匹配或超过部分 reference。Evolution 能带来提升，但 hidden-task transfer 与跨 runtime-model transfer 都不稳定。

第三篇是 **Harness-of-Harness: Multi-Day Autonomous Software Development with Continual Improvement**。

HoH 在现有 coding harness 外增加 planning、coding、testing 和 independent evaluation 的迭代控制层。它把开发拆成小而可验证的 increment，逐步开放 deliverable、tool 与 skill，并维护 versioned project history。三个 benchmark 与三组 harness-model pairing 上，三轮迭代平均获得 52.25% relative gain；一次超过 70 轮的 multi-day run 最终生成了可玩的 FPS 游戏。

论文：<https://arxiv.org/abs/2609.01572>

另外两篇：<https://arxiv.org/abs/2609.01437>、<https://arxiv.org/abs/2609.01481>
