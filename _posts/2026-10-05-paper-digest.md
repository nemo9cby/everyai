---
title: "Paper Digest: 2026-10-05"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, SFT, Post-training, Agent Harness]
---

今天最值得看的 paper 是 **Scaling Trajectories for Complex Tasks through Recursive Self-Rewrite**。

它抓住了 code-agent training 里一个很现实的问题：强 harness 能让模型完成更多任务，但成功轨迹里混入了 controller intervention、workflow convention、stopping rule 等外部能力。把这些轨迹直接拿去 SFT，模型可能学会依赖训练时的脚手架，部署到普通 harness 后就失效。

作者提出 Recursive Self-Rewrite，简称 RSR。核心流程可以压缩成一句话：让 specialized harness 帮模型发现解法，再让模型在 deployment harness 里重新练会一次。

## 三个 harness 扩大经验覆盖面

作者用同一个 Qwen-3.8-27B，在 Terminus 2、StateM 和 Recursive Self-Reflect Terminus 三种 harness 下处理约 3,000 个 terminal tasks。

三个 harness 一共发现了 2,001 条成功轨迹，覆盖 759 个不同任务。最强的单一 harness 只能覆盖 565 个任务，三者取 union 后，多解决了 194 个任务，相对增加 34.3%。

其中 288 个任务只能被某一个 harness 解出。这说明 harness 提供的 planning、progress tracking、recovery 和 validation 机制会激活不同的行为模式。训练数据的上限，也会被单一 harness 的能力边界卡住。

## 把成功轨迹重写成可部署能力

RSR 没有直接蒸馏原始轨迹。它让同一个 base model 扮演三个角色：

1. Planner 从成功轨迹中提炼 runbook，包括目标状态、关键步骤、检查方法、恢复策略和常见坑。
2. Critic 检查 runbook 里有没有 verifier leakage、solution leakage 或 harness-specific 信息。发现泄漏后，带着原因退回 planner 重写。
3. Executor 拿着合格 runbook，在 fresh sandbox 和通用 Terminus 2 harness 下重新完成任务。

Runbook 只在生成时作为 private guidance，不会进入最终训练数据。训练集只保留公开的 task instruction、environment observation 和 model response。每条重写轨迹还要重新通过 task verifier，并过滤无法从任务或环境推导出来的可疑值。

作者为每条 source success 最多生成四个 runbooks，每个 runbook 再做四次 fresh execution。最终，2,001 条 source trajectories 被扩成 11,094 条 verified trajectories。

这里的关键设计是 fresh re-execution。模型需要在目标 harness 里真实地观察环境、执行动作、处理失败和完成验证。成功经验因此被转成了与部署接口兼容的新轨迹。

## 直接 SFT 会把 harness 的坏习惯一起学进去

实验比较 Base、Direct SFT 和 RSR 三种模型。Direct SFT 会清理明显的 harness 插入文本，然后直接学习原始成功轨迹；RSR 学习重新执行并验证过的轨迹。

RSR 的 Pass@3 结果是：

- Terminal-Bench 2：57.0% → 74.2%
- Terminal-Bench 3：0.0% → 9.5%
- Terminal-Bench 4：1.5% → 9.1%
- Terminal-Bench Hard：39.0% → 63.0%
- Software Terminal-Bench：3.0% → 6.0%

Direct SFT 在部分 benchmark 上有效，却让 Terminal-Bench 2 从 57.0% 降到 53.4%。作者检查失败轨迹后发现，模型会重复相同动作，进入无法推进的循环。原始数据里一些超过 100 steps 的成功轨迹依赖 harness 提供持续控制，27B 模型只看到轨迹，很难把这种外部支持完整内化。

RSR 在同一个 benchmark 达到 74.2%。这组对比很有价值：轨迹通过 verifier，只能说明 model-harness system 当时成功了，不能自动证明这条轨迹适合目标模型和目标 harness 学习。

## 对 code-agent post-training 的启发

这篇论文给出了一条很实用的数据 pipeline：

1. 用多个 harness 做 experience discovery，扩大可解任务覆盖面
2. 把成功过程压缩成 procedural guidance
3. 对 guidance 做 solution、verifier 和 harness leakage audit
4. 在 deployment harness 和 fresh environment 里重新执行
5. 只保留重新通过 verifier 的公开 interaction history
6. 用 rewritten trajectories 做 SFT

真正需要评估的是 rewriting 的经济账。每条 source success 最多触发 16 次新执行，rollout、sandbox 和 verifier 成本都不低。最有意义的 ablation 应该在相同 rollout budget 下比较 direct SFT、简单 trajectory normalization 和 RSR，观察新增 verified successes 是否足以覆盖数据生成成本。

论文也有清楚的边界。Terminal-Bench Hard 和 Software Terminal-Bench 来自作者自建数据，后者只有 100 个任务。Long-Horizon Terminal-Bench 上，process reward 从 0.21 提高到 0.29，但三种模型都没有完成 46 个任务中的任何一个。这里展示的是更好的 partial progress，距离真正攻克 long-horizon tasks 还有很长一段路。

## 今天另外两篇相关论文

**WebUIProof** 用 UI-agent 在 headless browser 里执行 plan-act-observe 测试，评估生成网页在真实交互之后是否满足 specification。八个商业模型经常生成能渲染、却无法正确交互的页面。作者进一步把 executable interaction tests 作为 RL reward，提高 Qwen2.5-14B 和 MIMO-7B 的 functional completion，并减少 build failure。

**WEFT** 把 tool-use post-training 看作 environment、task、harness 和 evaluator 的整体工程。它用 execution evidence 定位失败组件，并引入 prefix-preserving sampling、atomic-turn credit assignment，以及支持并发 rollout 状态隔离和恢复的 MegaMCP。这个方向对大规模 agent training 的价值很直接：reward quality 和 execution infrastructure 必须一起扩展。

论文：<https://arxiv.org/abs/2610.02826>

另外两篇：<https://arxiv.org/abs/2610.02617>、<https://arxiv.org/abs/2609.36887>
