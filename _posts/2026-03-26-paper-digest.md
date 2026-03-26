---
title: "Paper Digest: 2026-03-26"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Research]
---

今天最值得看的 paper，不在于谁又把 reasoning benchmark 推高了半个点，而在于有人把一个大家都隐约感觉到的问题说透了：post-training 可能会把模型训练得更像一个“回答机器”，同时更不像一个会犹豫、会修正、会在未知问题前刹车的思考者。

## 今日 3 篇精选

### 1) Why Does Self-Distillation (Sometimes) Degrade the Reasoning Capability of LLMs?
- 链接: https://arxiv.org/abs/2603.24472
- 摘要速读: 这篇 paper 盯住了一个很关键、也很容易被忽略的 post-training 问题。self-distillation 往往能让模型输出更短、更整洁，看起来像是更“高效”了，但在数学推理任务上，性能反而可能明显下降。作者给出的解释很有意思：distillation 过程会压制模型对不确定性的显性表达，也就是 paper 里说的 epistemic verbalization。模型更少说“我不确定”“我需要再检查一下”这类中间信号，表面上更流畅，实际上在 OOD 问题上更容易一路自信地走错。
- 为什么重要: 这篇 paper 的价值，在于它不是单纯告诉你“某方法掉分了”，而是指出掉分发生在什么行为层。很多 post-training 方法追求的，其实是更短、更像 expert、更少噪音的回答。但 reasoning 的一部分，可能恰恰需要保留那些看似啰嗦、其实是校准信号的内容。对做蒸馏、SFT、RL 的人来说，这是个很硬的提醒。

### 2) T-MAP: Red-Teaming LLM Agents with Trajectory-aware Evolutionary Search
- 链接: https://arxiv.org/abs/2603.22341
- 摘要速读: 这篇 paper 关心的是 agent red-teaming 到底该怎么做。过去很多安全测试，本质还是 single-turn jailbreak，看模型会不会吐出不该说的话。T-MAP 认为这远远不够，因为 agent 的真实风险来自 multi-step tool execution，尤其是在 MCP 这类工具生态里。作者用 trajectory-aware evolutionary search 去找更有效的 adversarial prompts，不只是绕过安全规则，还要真的把有害目标通过工具链执行出来。
- 为什么重要: agent safety 终于开始从“文本安全”走向“执行安全”。这个转向很关键。危险不只在模型说了什么，更在模型做了什么。对所有在做 coding agent、tool-use system、MCP infra 的团队，这类工作都会越来越重要。

### 3) CUA-Suite: Massive Human-annotated Video Demonstrations for Computer-Use Agents
- 链接: https://arxiv.org/abs/2603.24440
- 摘要速读: 这篇 paper 更像数据基础设施。作者认为 computer-use agents 迟迟上不去，一个核心瓶颈是缺少连续、高质量的人类桌面操作视频，而不是缺少更多静态截图。CUA-Suite 给出了一整套资源，包括 55 小时的连续录屏、鼠标轨迹、reasoning annotation，以及补充的 grounding 和 evaluation 数据。
- 为什么重要: 很多时候，真正改变一个方向的不是某个 clever trick，而是训练数据终于变得像这个任务本身。对 GUI agent 来说，连续交互轨迹天然比稀疏截图更接近“行动中的智能”。如果 computer-use 这条线还会继续加速，这类 dataset 大概率是重要燃料。

## 可能感兴趣

### 4) UI-Voyager: A Self-Evolving GUI Agent Learning via Failed Experience
- 链接: https://arxiv.org/abs/2603.24533
- 亮点: 用 failed trajectories 做自演化训练，里面的 RFT + GRSD 组合值得单独跟。AndroidWorld 81.0% Pass@1 也很亮眼。

### 5) EVA: Efficient Reinforcement Learning for End-to-End Video Agent
- 链接: https://arxiv.org/abs/2603.22918
- 亮点: planning-before-perception + SFT/KTO/GRPO 三阶段训练，偏 RL 和 agentic video understanding，适合 thesis 向继续深挖。

## 一句话结论
今天最强的研究信号是，post-training 优化不能只盯着“更短、更稳、更像标准答案”。如果连不确定性也一起优化掉，模型可能会变得更顺滑，却更脆。