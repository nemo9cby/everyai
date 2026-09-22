---
title: "Paper Digest: 2026-09-22"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Agent Harness, Distillation, Post-training]
---

今天最值得看的 paper 是 **Harness-Zero: Harness Distillation via Agent-as-Harness**。

Agent 的能力经常来自模型外部：system prompt、tool schema、control flow、memory、context management，以及各种为特定任务手工打磨的 scaffold。一个强 harness 可以显著提高成功率，但也带来部署负担。不同 domain、不同 instance，甚至不同 model 可能各自需要一套配置，线上系统最终要维护越来越多的 routing 和 specialized harness。

Harness-Zero 提出一个很直接的问题：这些由 harness 引出的行为，能否在训练阶段写进模型参数，让部署时只保留一套固定、简单的 target harness？

论文给出的答案是 agent-as-harness。实验中，移除 specialized harness 之后，模型的 macro-average task success 从 23.3% 提升到 44.3%。这个结果还超过了 base model 继续挂载 specialized harness 时的 41.7%。在 knowledge work、tool use 和 science 三个 domain 的 28 种行为模式上，平均恢复率达到 82.3%。

## Harness 很强，但无法直接当 teacher

普通 distillation 假设 teacher 与 student 的输出空间大致一致。Agent harness 打破了这个假设。

一个 optimized harness 可能拥有 target harness 没有的工具，能看到更多中间状态，或通过额外的 control flow 反复检查答案。直接复制它的 trajectory，student 在部署环境里可能根本无法执行。即使两个 harness 都能完成同一项任务，它们的 action space 和 available information 也可能完全不同。

Harness-Zero 在 teacher 与 student 之间加入一个 harnessing agent。它观察 specialized harness 给出的指导，也看到 student 原本想执行的动作，然后在 target harness 的 action space 内修改 student response。修正后的动作会被实际执行，环境结果再进入训练 trajectory。

这一步很关键。Specialized harness 提供 privileged guidance，最终 supervision 仍然落在生产环境可执行的动作上。模型学到的是如何在 target harness 中复现有效行为，而非机械模仿一套部署时不存在的接口。

## Agent-as-harness 如何生成训练数据

整个过程可以拆成四步：

1. Student 在固定 target harness 中提出下一步 response。
2. Harnessing agent 读取 specialized harness 的指导并检查 student response。
3. Harnessing agent 在 target action space 内纠正 response，再交给环境执行。
4. 成功 trajectory 被收集，用于 fine-tuning student。

作者把这种方法与 code-as-harness 做了比较。后者依赖预先写好的规则或程序，把 specialized harness 的策略翻译到 target harness。对于 frontier LLM，agent-as-harness 表现更好。原因并不神秘：很多 harness 行为包含语义判断、动态规划和上下文依赖，很难完整压缩成静态规则。

训练完成后，harnessing agent 和 specialized harness 都可以移除。线上只运行 fine-tuned model 与统一 target harness。

## 23.3% 到 44.3%

论文覆盖 knowledge work、tool use 和 science tasks。最醒目的结果有三个：

- Base model 在 target harness 下的 macro-average success 为 23.3%
- Base model 挂载 specialized harness 后达到 41.7%
- Harness-Zero fine-tuning 后移除 specialized harness，达到 44.3%

最后一个数字值得仔细看。Distilled model 超过了直接使用 specialized harness 的 base model。这说明 correction trajectory 不只复制了外部 scaffold 的表面动作。多轮训练数据可能让模型吸收了更稳定、更容易复用的策略，也减少了运行时 harness 与模型之间的协调损耗。

作者还定义了 28 种 harness-induced behavior pattern，例如更好的信息搜集、tool selection、verification 与恢复策略。Harness-Zero 平均恢复其中 82.3%。这个分析比单一 success rate 更有价值，因为它开始回答一个 post-training 团队真正关心的问题：外部系统究竟教会了模型什么？

## 对 agent post-training 的意义

Agent 团队通常同时维护两套迭代循环。一套优化 model weights，另一套优化 prompts、tools 和 orchestration。后者实验速度快、可解释性强，也更容易针对具体 benchmark 获得收益；前者部署路径更统一，推理时开销也更可控。

Harness-Zero 给两套循环增加了一条数据通道：先用 harness 快速发现有效行为，再把这些行为转成 executable correction trajectories，用 SFT 写进模型。这里的训练样本天然包含几个重要字段：

- student 原始 proposal
- specialized harness 的 guidance
- harnessing agent 的 correction
- target harness 中的实际执行结果
- correction 对应的 behavior pattern
- token、latency 与 tool-call cost

这些记录可以支持更细的训练诊断。团队能够区分“模型已经会做”“外部 harness 暂时补足”“经过 distillation 后可以独立完成”三种状态，也能计算每种 harness mechanism 的 recovery rate。

## 风险也被写进了模型

外部 harness 的优点之一是容易观察和修改。Prompt 或 control flow 出问题，可以单独回滚。行为进入 weights 后，服务更简单，调试难度却可能上升。

因此，实际采用 Harness-Zero 时至少要监控四件事：

- Teacher harness 的错误是否被大规模固化
- Correction trajectory 是否覆盖失败恢复和 OOD 情况
- Model upgrade 或 tool schema 变化后，内化行为是否仍然有效
- 简化 runtime 所节省的成本，是否高于数据生成和 fine-tuning 成本

论文的 28-pattern recovery evaluation 提供了一个起点。生产系统还需要 regression suite，把每一种准备 internalize 的机制单独测量，并保留 provenance。否则线上看到一个动作时，很难判断它来自 base capability、distilled harness behavior，还是新的意外泛化。

## 今天另外两篇相关论文

**RRSI: Regularized Recursive Self-Improvement of Agent Harnesses** 研究 harness 的自动演化。它通过 annealed edit budget、探索奖励、critic 和 pruner 减少 benchmark overfitting。在八个 coding、workspace 与 engineering-design benchmarks 上，RRSI 的 in-distribution gain 最高达到 14.1 points，五个 OOD benchmarks 最高提升 4.7 points，同时比未正则化的 harness evolution 少用 30% policy tokens。

**One to More, More to One** 聚焦 software-engineering agent 的 category see-saw。作者训练不同 task category 的 experts，交替进行 Agentic-miniRL、verified trajectory repair SFT 和 curriculum refresh，再用 multi-teacher on-policy distillation 合并成一个 student。最终模型在 Pro-618 达到 58.04%，在 SWE-bench Multilingual 达到 59.00%。

三篇放在一起，会形成一条很完整的 agent improvement pipeline：RRSI 搜索更能泛化的 harness，Harness-Zero 把 harness 行为转成训练数据，category-aware distillation 再把多个 specialized policies 合并进单一模型。

Harness-Zero 最值得复现的部分，是跨 action space 的 correction data generation。对做 code agent 或 tool agent post-training 的团队来说，harness 可以成为 teacher，但 supervision 必须落在真实部署环境能够执行和验证的 trajectory 上。

论文：<https://arxiv.org/abs/2609.24974>

另外两篇：<https://arxiv.org/abs/2609.24972>、<https://arxiv.org/abs/2609.23377>
