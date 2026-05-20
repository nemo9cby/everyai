---
title: "Paper Digest: 2026-05-20"
categories: [Paper Digest]
tags: [AI, Agents, Computer Use, Tool Use, Post-Training]
---

今天最值得看的 paper，我会给 **OpenComputer**。

这篇 paper 处理的是 computer-use agent 里一个很基础、也很麻烦的问题：我们到底怎么判断 agent 真的完成了任务。

很多现有 benchmark 看的是最终截图、最终文本，或者干脆让 LLM 当裁判。但真实软件操作里，成功常常藏在细粒度状态里。一个 spreadsheet 公式有没有填对，一个 IDE 里的文件有没有改到正确位置，一个邮件草稿是不是在正确账户下保存，这些事情只看表面输出，很容易误判。

OpenComputer 的价值，在于它把评测往“可验证的软件世界”推近了一大步。

作者做了四层东西：针对真实应用的 state verifier、会根据执行反馈改进的 verification layer、可机器检查的 desktop task 生成流程，以及能记录完整轨迹并计算 partial credit reward 的 evaluation harness。当前版本已经覆盖 33 个桌面应用和 1000 个任务，范围包括浏览器、office、开发环境、文件管理和沟通软件。

我觉得这篇最有分量的地方，是它没有靠更花哨的 agent loop 来制造乐观结果，而是先把尺子做得更硬。

结果也很说明问题。作者发现 hard-coded verifier 比 LLM-as-judge 更贴近人工判定，尤其是在任务依赖细粒度应用状态的时候。更扎心的是，当前 frontier agent 在这种更扎实的评测里，离稳定完成 end-to-end software automation 还有明显距离。也就是说，很多“看起来会用电脑”的能力，一旦进入可审计的状态检查，水位会立刻掉下来。

这对 agent 圈子其实是好消息。

因为真正有价值的进步，必须建立在更可信的反馈上。否则训练、评测、宣传，全都容易围着一层模糊表象打转。OpenComputer 给出的方向很清楚：如果我们想认真做 computer-use agent，就得把环境状态、任务验证、partial reward 和轨迹审计一起纳入系统设计。

今天另外两篇也值得一起看。

**EnvFactory** 很像 OpenComputer 在训练侧的对应物。它自动探索和验证 tool environment，再生成 grounded 的多轮轨迹去做 SFT 和 agentic RL。这个思路很实在，重点放在“可执行环境从哪里来”，而不是只堆更多 synthetic trace。

**GoLongRL** 则更偏 long-context post-training。它把 long-context RLVR 从单一 retrieval 路径拉回 capability taxonomy，开源了 23K 样本、构建流程和训练代码。对做数据设计和 reward alignment 的人，这篇很有参考价值。

如果把今天这三篇放在一起看，我觉得最清楚的一点是：**agent 的上限，很大程度上取决于环境和反馈能不能做实。** OpenComputer 让评测更硬，EnvFactory 让训练环境更可扩展，GoLongRL 让长上下文奖励设计更贴近真实能力。

所以今天最推荐先读 `2605.19769`。它不会给人一种“agent 已经快会像人一样用电脑”的幻觉，反而把问题摊得更清楚。这样的 paper，通常更值得认真看。

## 今日 3 篇精选

### 1) OpenComputer: Verifiable Software Worlds for Computer-Use Agents
- 链接: https://arxiv.org/abs/2605.19769
- 摘要速读: 用 app-specific verifier、task synthesis 和 auditable partial-credit reward，把 computer-use agent 评测建立在真实软件状态之上。
- 为什么重要: 它让大家更准确地看到，当前 agent 在真实软件自动化里到底卡在哪里。

### 2) EnvFactory: Scaling Tool-Use Agents via Executable Environments Synthesis and Robust RL
- 链接: https://arxiv.org/abs/2605.18703
- 摘要速读: 自动构建并验证可执行 tool environment，再生成 grounded 多轮轨迹做 SFT 和 RL。
- 为什么重要: 它把 tool-use agent 的训练瓶颈，直接落在环境供给和轨迹质量上。

### 3) GoLongRL: Capability-Oriented Long Context Reinforcement Learning with Multitask Alignment
- 链接: https://arxiv.org/abs/2605.19577
- 摘要速读: 开源 23K long-context RLVR 样本和完整 pipeline，并用 multitask reward reweighting 改善训练稳定性。
- 为什么重要: 它给 long-context post-training 提供了更可复用的数据和 reward 设计框架。

## 一句话结论
今天最强的研究信号是，**agent 要想变得可靠，环境必须更可执行，反馈必须更可验证。** `2605.19769` 值得先读。