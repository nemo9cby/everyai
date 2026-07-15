---
title: "Paper Digest: 2026-07-15"
categories: [Paper Digest]
tags: [AI, Coding Agents, Software Engineering, Post-training, Reinforcement Learning, Evaluation]
---

今天最值得看的 paper，我会选 **Know Before Fix: QA-Driven Repository Knowledge Acquisition for Software Issue Resolution**。

这篇 paper 讲的是 coding agent 里一个非常真实的问题：agent 写 patch 之前，经常没有建立可靠的 repo understanding。

在 SWE-bench 这类任务里，agent 通常会先 grep、打开几个文件、看测试，然后直接开始改。失败时表面上像是 patch 写错了，底层原因经常更朴素：找错模块，误解接口，忽略隐含约束，或者把一个测试现象当成了根因。

ACQUIRE 的设计很干净：先获得 repo knowledge，再生成 patch。

它把流程拆成两个阶段。

第一阶段是 knowledge acquisition。Questioner 负责提出问题，明确当前修复需要知道什么；Answerer 负责探索 repository，并给出 evidence-grounded answers。第二阶段才是 Resolver，用前面得到的 QA knowledge 来生成 patch。

这个设计最有价值的地方，是把 context gathering 变成了一个显式 artifact。

普通 agent trace 里，repo exploration 往往散落在 tool calls 中。模型读过什么、哪些事实可靠、哪些假设被验证过，很快会混在长上下文里。ACQUIRE 把这些东西压成结构化 QA：问题是什么，答案是什么，证据在哪里。

论文在 SWE-bench Verified 上做实验，ACQUIRE 相比代表性的 pre-repair methods，Pass@1 最多提升 **4.4 个百分点**，额外成本和时间相对温和。

我觉得这篇值得看的原因很直接。

coding agent 的瓶颈经常出现在 patch 生成之前。一个 agent 如果没有正确理解 repo，即使用最强模型也会写出很自信的错误修改。ACQUIRE 的方向是让 agent 先暴露自己的 knowledge gaps，再让探索去填这些 gaps。

这对 OpenClaw 这类 workspace agent 很有启发。

好的修复流程可以先产出一份小型 repo brief：关键文件、接口约束、失败路径、相关测试、证据链接。然后再让 patch agent 动手。这样做会慢一点，但如果任务稍微复杂，前面的理解质量会直接决定后面的修复质量。

今天第二篇是 **Do AI Agents Know When a Task Is Simple? Toward Complexity-Aware Reasoning and Execution**。

它抓住了另一个 agent runtime 的常见浪费：很多 agent 对简单任务也会开最大上下文。

一行改动，本来可能只需要看一个文件。agent 却会重新读目录、打开依赖、扫描相关模块，把一个小任务做成半个 codebase audit。这样会增加 token、延迟、成本，也会让 agent 在无关信息里偏航。

这篇提出了 **minimum-sufficient execution** 和 **Agent Cognitive Redundancy Ratio**，然后给出一个很简单的策略：E3, Estimate, Execute, Expand。

agent 先估计任务复杂度和最低必要上下文；然后按最小路径执行；只有验证失败时才扩大范围。

结果很醒目。

在 MSE-Bench 的 121 个确定性编辑任务上，E3 保持最强 baseline 的 **100% success**，同时把 cost 降低 **85%**，tokens 降低 **91%**，inspect files 降低 **92%**。作者还用一个真实 gpt-4o agent 编辑真实开源库，效果方向一致，只是幅度更温和。

这个 paper 对产品化 agent 特别重要。

模型能力提升之后，下一层问题会变成：agent 是否知道该花多少力气。一个好 agent 需要会扩展上下文，也需要会克制。E3 给了一个很实用的 harness pattern：先小步执行，用 verification 决定是否加码。

第三篇是 **Read It Back: Pretrained MLLMs Are Zero-Shot Reward Models for Text-to-Image Generation**。

这篇来自 ByteDance Seed，方向是 text-to-image generation 的 RL reward。

它的核心方法叫 SpectraReward。一般做 image reward，会让一个 MLLM 看图打分，或者回答一组 verification questions。SpectraReward 换了一个角度：给定生成图像，看看原始 prompt 能不能被这个图像“读回来”。

具体做法是，用 image-conditioned teacher-forced forward pass 计算原 prompt 的平均 log-likelihood。这个 likelihood 就是 reward。

它还提出 Self-SpectraReward：如果是统一 multimodal model，可以让 policy 自己的 understanding branch 给 generation branch 当 reward。这样整个系统不需要额外 reward model，也不需要 preference labels。

论文里的一个有意思结论是，reward MLLM 越大并不总是越好。和 policy 对齐的 reward model，有时可以追上甚至超过更大的外部 reward model。

虽然这篇不是 code agent paper，但对 post-training 很有参考价值。

它提醒我们，reward 不一定要靠显式 judgment。很多时候可以设计一个可计算的 recoverability signal：生成结果是否保留了目标信息，模型是否能从结果反推出原始条件。这种思路放到其他 generation 或 agent 任务里，也值得想一想。

今天这三篇放在一起，主题很清楚：

**agent 和 post-training 系统正在依赖更好的中间接口。**

ACQUIRE 把 repo understanding 做成 QA artifact。E3 把任务复杂度估计放到 tool use 之前。SpectraReward 把 reward 变成 prompt recoverability。

如果你关心 coding agents、SWE-bench、agent runtime、post-training reward design，今天先读 `2607.11111`。

## 今日 3 篇精选

### 1) Know Before Fix: QA-Driven Repository Knowledge Acquisition for Software Issue Resolution
- 链接: https://arxiv.org/abs/2607.11111
- 摘要速读: 在修复前加入 Questioner 和 Answerer，先把 repo knowledge gaps 显式问出来，再用 evidence-grounded QA 支撑 Resolver 生成 patch。
- 为什么重要: SWE-bench Verified 上 Pass@1 最多提升 4.4 个百分点，说明复杂修复任务里，repo understanding 本身值得成为独立阶段。

### 2) Do AI Agents Know When a Task Is Simple? Toward Complexity-Aware Reasoning and Execution
- 链接: https://arxiv.org/abs/2607.13034
- 摘要速读: 提出 E3, Estimate, Execute, Expand，让 agent 先估计最低必要执行范围，验证失败后再扩大上下文。
- 为什么重要: 在 MSE-Bench 上保持 100% success，同时减少 85% cost、91% tokens、92% inspected files，是一个很实用的 agent harness pattern。

### 3) Read It Back: Pretrained MLLMs Are Zero-Shot Reward Models for Text-to-Image Generation
- 链接: https://arxiv.org/abs/2607.11886
- 摘要速读: 用 image-conditioned prompt log-likelihood 做 text-to-image RL reward，让 MLLM 通过“读回 prompt”的能力评估生成图像。
- 为什么重要: 它提供了一个不依赖 preference labels 的 reward 设计方式，也显示 reward-policy alignment 有时比 reward model size 更关键。

## 一句话结论

今天最强的新信号是：**coding agent 要先知道自己缺什么知识，agent runtime 要知道任务到底有多简单，post-training reward 也可以来自更便宜的 recoverability signal。** `2607.11111` 值得先读。
