---
title: "Paper Digest: 2026-05-14"
categories: [Paper Digest]
tags: [AI, Agents, Post-Training, Infrastructure]
---

今天最值得看的 paper，我会给 **MinT: Managed Infrastructure for Training and Serving Millions of LLMs**。

这篇 paper 的价值很直接，它讨论的已经不是“怎么再训出一个更强的 LoRA policy”，而是**当你真的开始大规模做 post-training 之后，policy 生命周期本身会不会成为系统瓶颈。**

作者把问题定义得很清楚。现实里的 RL/post-training 系统，常常会在少量非常昂贵的 base model deployment 上，持续产出大量 policy。如果每次都把 adapter merge 成完整 checkpoint，再去 rollout、评估、上线、回滚，这个流程很快就会变得又重又慢。

MinT 的核心做法，是把 **LoRA adapter revision** 当成真正的运营单元。base model 常驻，训练和服务系统围绕轻量 adapter 流转。paper 里把这个思路拆成三个层面。

第一层是 **Scale Up**。他们把 LoRA RL 扩到 frontier-scale dense 和 MoE 模型上，连 MLA、DSA 这类 attention path 也覆盖进去，目标很明确，就是让这种 adapter-first 的管理方式能真的接住 1T 级别总参数规模。

第二层是 **Scale Down**。这里最实用。因为传递的是导出的 LoRA adapter，而不是完整权重，policy handoff 的代价会明显下降。paper 报告的数字也很有说服力，4B dense 模型上 step transfer 降了 18.3x，30B MoE 上也有 2.85x。对并发多 policy GRPO，他们也给出了 wall time 改善。

第三层是 **Scale Out**。这部分最像真正的基础设施论文。他们把“可寻址的 policy catalog”和“CPU/GPU 上当前激活的工作集”分开，让系统可以维护百万级 policy catalog，同时只把当前需要的 adapter wave 拉进在线执行路径。这个抽象很对，因为真实系统里最大的浪费，常常就来自把长期存档和在线热路径混在一起。

我觉得这篇 paper 最有意思的地方，在于它让人重新意识到：**post-training 的竞争，已经越来越像 systems competition。**

很多时候，算法能不能跑起来只是第一关。真正决定速度、成本、稳定性、回滚能力、实验吞吐和线上切换效率的，是 policy artifact 怎么流动，怎么编址，怎么装载，怎么共享底座，怎么从训练路径进入服务路径。

对做 RLHF、GRPO、LoRA serving、multi-policy deployment 的团队来说，这篇很值得看。因为它给的不是抽象愿景，而是一套比较完整的工程视角：

- policy 是否应该继续被当成完整 checkpoint 管理？
- adapter-only handoff 能把多少系统摩擦拿掉？
- 多 policy 训练和在线 serving，能不能共用同一套底座抽象？
- catalog scale 和 active working set，是否应该彻底分层？

如果这些问题开始变成主问题，那就说明 agent/post-training 这条线，已经从“能不能做”走到了“怎么规模化运转得更像基础设施”。这正是 MinT 最值得读的原因。

## 今日 3 篇精选

### 1) MinT: Managed Infrastructure for Training and Serving Millions of LLMs
- 链接: https://arxiv.org/abs/2605.13779
- 摘要速读: 把 LoRA adapter revision 当成核心运营单元，在常驻 base model 之上打通 rollout、update、export、evaluation、serving、rollback，目标是支撑百万级 policy catalog 与 frontier-scale RL/serving。
- 为什么重要: 它把 post-training 的瓶颈从模型本身，推进到了 policy 生命周期和系统编排层。

### 2) SkillOps: Managing LLM Agent Skill Libraries as Self-Maintaining Software Ecosystems
- 链接: https://arxiv.org/abs/2605.13716
- 摘要速读: 把 agent skill library 看成会积累技术债的软件生态，用 typed skill contract、层级图结构和低开销维护层，持续修复 utility、compatibility、risk、validation 等库级问题。
- 为什么重要: 它提醒我们，agent 的能力库本身也会腐化，库级维护会慢慢变成必要层。

### 3) Learning Agentic Policy from Action Guidance
- 链接: https://arxiv.org/abs/2605.12004
- 摘要速读: 用普通人类交互里产生的 action data 作为 plan-style guidance，帮助 agentic RL policy 穿过探索可达性障碍，再通过 mixed-policy training 把这些增益收回到 unguided policy。
- 为什么重要: 它给了一条比重型冷启动 SFT 更轻的 agentic RL 路线。

## 一句话结论
今天最强的研究信号是，**agent 体系的真正难点越来越往基础设施层下沉。** 从 LoRA policy 的生命周期管理，到 skill library 的维护，再到 agentic RL 的探索信号注入，关键问题都在往“怎么长期稳定运转”聚焦。`2605.13779` 值得优先读，因为它把这件事讲得最完整，也最接近现实系统压力。