---
title: "Paper Digest: 2026-05-25"
categories: [Paper Digest]
tags: [AI, Agents, Coding Agents, Skills, Post-Training]
---

今天最值得看的 paper，我会给 **SkillOpt**。

这篇 paper 讲的是 agent skills，但它真正有意思的地方，是把 skill 当成一种可以被训练的外部状态来处理。

很多 agent 系统里的 skill，本质上还是一份手写文档：告诉模型怎么做事、注意什么、调用哪些工具。它可以被人修改，也可以被模型总结，但常见流程都偏松散。写得好不好，往往靠经验和事后感觉判断。SkillOpt 的主张更硬一点：既然 skill 会影响 frozen agent 的行为，那就应该像优化参数一样，用 rollout、score、validation gate 和稳定的 edit budget 去优化这份文本。

它的机制很直接。一个 optimizer model 读取 scored rollouts，然后对单个 skill 文档做有限的 add / delete / replace 编辑。每次编辑只有在 held-out validation score 真的提高时才被接受。系统还维护 textual learning-rate budget、rejected-edit buffer 和 epoch-wise slow/meta update，用来避免 skill 被一次反馈带偏。

最关键的是，部署时没有额外 inference-time model call。训练发生在 skill 文档上，最终 agent 只消费优化后的 skill。

结果很强。作者在 6 个 benchmark、7 个 target model、3 种 execution harness 上做评估，包括 direct chat、Codex agentic loop 和 Claude Code。SkillOpt 在全部 52 个 model / benchmark / harness 组合里 best or tied。GPT-5.5 上，它让 no-skill baseline 平均提升 +23.5 points；放进 Codex loop 里提升 +24.8；放进 Claude Code 里提升 +19.1。

这个结果有一个很实用的含义：agent 能力不只来自 base model。模型外面那层 skill、tool workflow、memory artifact，如果能被系统化优化，也能带来很大的行为提升。

这对 coding agent 尤其重要。现在大家会反复讨论模型本身会不会改代码、会不会跑测试、会不会做 long-horizon planning。SkillOpt 提醒我们，agent 的外部操作规程也可以成为可优化对象。一个更好的 skill，可能让同一个 frozen model 在 Codex 和 Claude Code 里都表现更好，还能跨模型规模迁移。

今天另外两篇也值得一起看。

**From Patches to Trajectories** 更偏 SWE-agent SFT。它指出，直接模仿长 teacher trajectory 会把有效调查步骤和无用循环一起教给 student。作者用 developer reference patch 作为 privileged information，反推出 latent process graph，再从 blinded teacher continuations 里筛出最短、最有效、不会泄漏答案的 trajectory segment。只用 1.8K curated SWE-Gym instances，就能在 SWE-bench Verified 上提高最多 10.8 Pass@1，同时降低约 15% inference cost。

**HINT-SD** 则看 long-horizon agent distillation。它用完整 trajectory hindsight 找到真正导致失败的 action span，只在这些位置做 feedback-conditioned distillation。这个思路比每一步都生成反馈更省，也更贴近失败原因。在 BFCL v3 和 AppWorld 上，它比 dense per-turn feedback baseline 最多提升 18.8%，训练 step 还快 2.26 倍。

把今天这三篇放在一起看，主线很清楚：agent 的训练和优化正在落到模型周围的结构上。skill 可以优化，SWE trajectory 可以用 patch 信息精修，long-horizon failure 可以只监督关键 action span。

所以今天先读 `2605.23904`。它和真实 coding-agent workflow 距离很近，也很适合用来思考 OpenClaw / Codex 这种系统里的 skill layer 到底应该怎么进化。

## 今日 3 篇精选

### 1) SkillOpt: Executive Strategy for Self-Evolving Agent Skills
- 链接: https://arxiv.org/abs/2605.23904
- 摘要速读: 用 validation-gated text-space optimization 训练 agent skill 文档，让 frozen agent 在不同 harness 里稳定提升。
- 为什么重要: 它把 skill 定义成了可优化、可迁移、可验证的 agent 外部状态。

### 2) From Patches to Trajectories: Privileged Process Supervision for Software-Engineering Agents
- 链接: https://arxiv.org/abs/2605.21996
- 摘要速读: 用 reference patch 作为 privileged information，筛选更有效、更短的 SWE-agent SFT trajectories。
- 为什么重要: 它把 coding-agent 训练里的“过程监督质量”直接压到 SWE-bench 这类任务上。

### 3) HINT-SD: Targeted Hindsight Self-Distillation for Long-Horizon Agents
- 链接: https://arxiv.org/abs/2605.17873
- 摘要速读: 用 trajectory hindsight 找到失败相关 action span，只在关键位置做 self-distillation。
- 为什么重要: 它让 long-horizon agent training 的反馈更聚焦，也更高效。

## 一句话结论
今天最强的研究信号是：**agent 的外部工作层正在变成可训练对象。** `2605.23904` 值得先读。
