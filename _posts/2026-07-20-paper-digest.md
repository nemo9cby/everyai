---
title: "Paper Digest: 2026-07-20"
categories: [Paper Digest]
tags: [AI, Reasoning, Reinforcement Learning, Post-Training, Distillation, Agents]
---

今天最值得看的 paper，我会选 **Understanding Reasoning from Pretraining to Post-Training**。

它抓住了 reasoning model 研究里一个长期被切开的接口：pretraining 把模型带到哪里，会怎样决定后面的 RL 能走多远、走多快。

常见的 post-training 实验会固定一个 pretrained model，然后比较 SFT、RLVR、reward design 或 optimizer。这样能回答某个 RL recipe 有没有用，却很难回答另一个更基础的问题：换一个 pretraining 状态，同样一份 RL compute 还值多少钱？

作者用 chess 做了一套受控实验。模型规模从 5M 到 1B parameters，先在 human chess games 上 pretrain，再用 synthetic reasoning traces 做 SFT，最后在带 verifiable reward 的 chess puzzles 上做 RL。

这个选择很聪明。棋局有清楚的状态、动作和正确性判定，训练规模也允许作者系统地 sweep model size、pretraining tokens 和 RL compute。原本在通用 LLM 上难以控制的变量，终于可以放进同一张实验图里。

最重要的结果有两个。

第一，在给定 RL compute 下，post-RL performance 可以被 pretraining loss 很好地预测。pretraining 更充分的 checkpoint 不只起点更高，RL reward curve 的增长斜率也更好，而且这种斜率大致随 pretraining tokens 线性改善。

这意味着 pretraining quality 同时影响 post-training 的终点和学习效率。两套 RL recipe 的结果差异，有时可能来自进入 RL 之前的 model state，而非 RL algorithm 本身。

第二，RL 做的事情会随题目难度变化。

在 easy puzzles 上，RL 主要放大 SFT policy 已经偏好的正确动作。在 hard puzzles 上，RL 能把 SFT 后几乎没有概率的正确动作推出来。这个观察给“RL 只是 sharpen policy”补上了很重要的边界：面对困难任务，RL 还可能在可验证反馈下完成 capability elicitation。

作者随后在 math domain 上训练 1B language model，中心规律仍然成立：pretraining 更久的 checkpoint，最终 RL performance 更高，改善速度也更快。

这篇 paper 的价值在于，它把 pretraining、SFT 和 RL 放回同一条训练链路中。对真正设计 post-training experiment 的团队，pretraining loss 可能成为估算 RL return 的 planning signal；比较算法时，也需要更认真地匹配进入 RL 前的模型状态。

今天第二篇是 **On-Policy Delta Distillation**。

普通 on-policy distillation 让 student 在自己的 trajectory 上模仿 teacher distribution。问题在于，teacher distribution 混合了通用语言能力、instruction following、style 和 reasoning 等多种变化，student 会把整套分布都当作学习目标。

OPD² 提出一个很干净的信号：用 reasoning-tuned teacher 减去它在 reasoning instruction tuning 之前的 base model。这个 delta 捕捉 reasoning tuning 对 token probability 带来的改变，再把这部分变化作为 dense distillation reward。

数学、科学和 code reasoning 实验中，OPD² 一致优于 conventional OPD，并能用较短 post-training 获得强表现。最值得带走的是 teacher-minus-base 这个设计，它把“向 teacher 学”改写成“学习 teacher 因目标能力而发生的变化”。

今天第三篇是 **RESOURCE2SKILL: Distilling Executable Agent Skills from Human-Created Multimodal Resources**。

它把 tutorial videos、repositories、articles 和 reference artifacts 转成 executable agent skills，并组织成 hierarchical multimodal Skill Wiki。每个 skill 同时保存结构化文字、代码、视觉示例、metadata 和 provenance，agent 在 inference 时检索、组合，也能在线补齐缺失能力。

在七类 practical authoring domains 上，RESOURCE2SKILL 相比 no-skill agent 平均提升 11.9 percentage points，并在 28 个主要 model-domain 对比中赢下 26 个。

这项工作的意义很直接：大量高质量 human procedural knowledge 藏在视频和混合媒介教程里。把它们编译成带 provenance、可执行、可组合的 skill，会比每次临时塞一段 context 更容易积累长期 agent capability。

今天三篇 paper 给出三个互补信号：

1. reasoning RL 的收益受到 pretraining state 的系统性约束。
2. distillation signal 可以只传递 teacher 因 reasoning training 发生的变化。
3. agent capability 可以从人类的多模态流程资源中持续沉淀。

如果只读一篇，先读 `2607.16097`。它提供了一套少见的 full-pipeline controlled study，也给 post-training compute allocation 留下了可直接检验的预测。

## 今日 3 篇精选

### 1) Understanding Reasoning from Pretraining to Post-Training
- 链接: https://arxiv.org/abs/2607.16097
- 摘要速读: 用 chess 和 math 的受控实验连接 pretraining、SFT 与 RL，发现 pretraining loss 能预测 post-RL performance 和 RL improvement rate。
- 为什么重要: 它说明 RL 的回报与进入 post-training 前的 model state 紧密耦合，并揭示 RL 在 hard tasks 上可以激活 SFT 几乎没有表达的正确行为。

### 2) On-Policy Delta Distillation
- 链接: https://arxiv.org/abs/2607.15161
- 摘要速读: 用 reasoning-tuned teacher 与其 base model 的概率差作为 dense on-policy distillation signal。
- 为什么重要: 它试图隔离并迁移 reasoning tuning 真正带来的 policy update，避免 student 复制 teacher 的全部分布。

### 3) RESOURCE2SKILL: Distilling Executable Agent Skills from Human-Created Multimodal Resources
- 链接: https://arxiv.org/abs/2606.29538
- 摘要速读: 将视频、代码仓库、文章和参考素材编译为可检索、可组合、带 provenance 的 multimodal executable skills。
- 为什么重要: 它给 agent skill acquisition 提供了可扩展的数据来源和经验验证过的组织方式。

## 一句话结论

今天最强的信号是：**pretraining quality 同时决定 reasoning RL 的上限与学习速度，post-training algorithm 需要放回完整训练链路中评估。**
