---
title: "Paper Digest: 2026-04-24"
categories: [Paper Digest]
tags: [AI, Post-Training, RL, Coding Agents, LLM]
---

今天最值得看的 paper，是一篇把 **LLM 蒸馏为什么总像很多零散经验拼起来的黑箱工艺** 这件事，重新整理成一套更清楚 recipe 的工作。**Hybrid Policy Distillation for LLMs** 关心的，不是再换一个新名字去包装蒸馏，也不是只在某个窄 benchmark 上多拿几个点，而是一个更实在的问题，**如果你真想把强模型的能力更稳地迁到小模型上，目标函数、优化方式和数据来源到底该怎么搭配。**

这篇 paper 的好处，是它先把问题讲明白。过去看 LLM distillation，很多方法像在各自押注一套偏好。有人偏 forward KL，觉得 mode coverage 更重要。有人偏 reverse KL，觉得输出得更 sharp、更像 teacher 才有效。有人强调全离线数据，便宜、稳定、好并行。也有人强调得带一点 on-policy，不然 student 永远学的是别人的分布，不是自己真正会走到的状态。每条路都有道理，但很多 paper 最后的感觉还是 recipe 碎片很多，缺少一个统一视角。

HPD 做得漂亮的地方，在于它把这些选择拉回到一个统一 token-level 视角里，然后给出一个不那么教条的答案。它没有站队，而是承认这些设计各自有价值，于是提出 **Hybrid Policy Distillation**，把 forward 和 reverse KL 结合起来，再把 off-policy 数据和轻量近似的 on-policy sampling 拼到一起。你可以把它理解成一套更像工程系统而不是学派信仰的做法，哪里该保 coverage，哪里该更 mode-seeking，哪里该靠便宜的静态数据，哪里该补一点 student 自己会遇到的轨迹，作者都在试图系统化。

这也是它今天最值得看的原因。它代表了一种很健康的研究取向，**post-training 不是继续追求一个万能目标函数，而是承认能力迁移本来就是多目标平衡问题。** 数学推理、对话、代码任务，这三类分布差异很大，但 paper 里最有意思的信号是，同一套混合式 recipe 居然能在这些场景里都带来更稳的优化和更好的结果。这说明它碰到的可能不是某个 benchmark 的小技巧，而是 distillation 里更底层的结构性问题。

如果你关心 open model、student model、cost-performance tradeoff，这篇 paper 尤其值得看。现在很多系统层面的机会，早就不只是做出更大的 teacher，而是想办法把 teacher 的能力更干净地搬下来，搬到更便宜、更快、更可部署的模型上。HPD 的价值，就在于它试图把这件事从“玄学调参”往“可解释的 recipe design”推进一步。

## 今日 3 篇精选

### 1) Hybrid Policy Distillation for LLMs
- 链接: https://arxiv.org/abs/2604.20244
- 摘要速读: 作者把现有蒸馏方法拆成 divergence 方向、优化方式和数据来源三类设计选择，再提出 HPD，把 forward / reverse KL 和 off-policy / 轻量 on-policy sampling 组合起来，在 math reasoning、dialogue、code 上都取得更稳的结果。
- 为什么重要: 这篇 paper 最像一份真正能带回去改训练配方的东西。它不是在喊新口号，而是在认真回答，student model 到底该如何向 teacher 学。

### 2) VLAA-GUI: Knowing When to Stop, Recover, and Search, A Modular Framework for GUI Automation
- 链接: https://arxiv.org/abs/2604.21375
- 摘要速读: 这篇工作把 GUI agent 的常见失败拆成三个控制问题，什么时候该停，失败后怎么恢复，下一步不明确时怎么主动搜索。作者用模块化框架而不是单一大模型策略去解决。
- 为什么重要: 很多 agent demo 卡住的地方，恰好就在这三件小事上。把它们显式化，比盲目堆更强模型更像正路。

### 3) WebGen-R1: Incentivizing Large Language Models to Generate Functional and Aesthetic Websites with Reinforcement Learning
- 链接: https://arxiv.org/abs/2604.20398
- 摘要速读: WebGen-R1 用 RL 训练 7B 模型生成多页网站，奖励里同时看结构、功能执行结果和视觉效果，目标不只是代码能跑，还要页面真的可用、可看。
- 为什么重要: 它把 code generation 的 reward design 往更真实产品输出推进了一步。软件产物不只是通过测试，还要在交互和视觉层面成立。

## 一句话结论
今天最强的研究信号是，**post-training 正在从“选一个最优原则”走向“设计一套更平衡、更可落地的 recipe”。** `2604.20244` 值得重点看，因为它提醒人，蒸馏真正难的地方，不是 teacher 强不强，而是你能不能把不同目标函数和不同数据分布的优点，拼成一条更稳的能力迁移路径。
