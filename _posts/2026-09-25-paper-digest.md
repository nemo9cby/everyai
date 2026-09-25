---
title: "Paper Digest: 2026-09-25"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Code Agents, Reinforcement Learning, Post-training]
---

今天最值得看的 paper 是 **Rufus-Air: An Open LLM Post-Training Recipe**。

很多模型报告会公开最终 benchmark，却把真正决定复现成败的部分压缩成几句话：SFT 数据怎样混，RL prompt 怎样筛，rollout 被截断后怎样处理，推理和训练精度不一致怎么办，sandbox 挂掉算谁的错，每个 stage 又该接在哪个 checkpoint 后面。

Rufus-Air 给出了一条完整流水线。它从 GLM-4.5-Air-Base 出发，这是一个 106B 总参数、12B 激活参数的 MoE，依次经过八个阶段：

1. SFT
2. Reasoning RL
3. Coding RL
4. Instruction-Following RL
5. General Agent
6. Coding Agent
7. Search Agent
8. RLHF

最终模型在 SWE-bench Verified 上达到 65.6%，Terminal-Bench 2.1 为 42.7%，LiveCodeBench v6 为 76.4%，MCP-Atlas 为 45.3%，BrowseComp 为 37.1%。这几项都高于官方 GLM-4.5-Air post-trained release。

## SFT 是能力底座

Rufus-Air 的 SFT 包含 901 万样本、445 亿 raw tokens。Mask 掉 system、user 和 tool observation 后，真正进入 loss 的 assistant tokens 是 270 亿。

样本数量和训练 token 的分布差异很大。General Agent 占 39.6% 的样本，却只贡献 14.2% 的训练 tokens；Math 与 Coding Agent 合计只占 18.8% 的样本，却贡献 49.1% 的训练 tokens。Agent trajectory 和 reasoning trace 更长，只报 sample count 很容易误判实际训练权重。

这个 SFT checkpoint 已经在 AIME 25、AIME 26 和单轮 instruction following 上超过官方 post-trained 模型。作者把 SFT 当作 capability-building stage，后续 RL 用于收窄具体短板。

还有一个很诚实的结果：training loss 在第二、第三个 epoch 继续下降，held-out scores 在第一个 epoch 内已经基本走平。最终进入 RL 的是 plateau 中间的 step 3799，没有选择 loss 最低的最后一个 checkpoint。

## Prompt difficulty 本身就是 curriculum

Reasoning RL 使用 12.1 万道 math、science 和 puzzle prompts。作者先用强 teacher 做 solvability check，过滤掉没有任何成功样本的坏题；再根据当前 policy 的通过率保留 `0 < success rate <= 0.8` 的题目。

全对的题几乎没有 gradient，全错的题也没有 group-relative signal。真正有效的是 policy 偶尔能解出来、但还没有掌握的区间。随着模型进步，原本全错的题会进入这个区间，形成自动 curriculum。

这条经验贯穿 Reasoning RL、Coding RL 和 Instruction-Following RL。论文对 post-training 的核心判断也很直接：模型已经有较强 SFT 底座以后，数据筛选通常比发明新的 RL objective 更重要。

## Reward 可靠性决定 stage 顺序

八个阶段大体按照 reward 从硬到软排列。

Reasoning RL 使用答案匹配和 Python checker，Coding RL 使用 execution tests。General Agent、Coding Agent 和 Search Agent 随后进入真实工具环境。开放式偏好的 RLHF 放在最后，减少模型在可被 reward hacking 的信号上持续优化的时间。

这个顺序也提醒了一件容易被忽略的事：后面的 stage 会继续更新同一套参数，早期能力并不会自动保存。Rufus-Air 因此让每个 stage 都和它实际接收的 checkpoint 比较，并同时监控其他能力的回退。

## 64K rollout cap 吃掉了 Coding RL 的训练信号

Coding RL 的一个细节很值得记住。

在 64K response budget 下，11% 到 28% 的 rollout 撞到长度上限。这些样本被截断后需要 mask loss，大量昂贵生成因此没有进入 policy update。把上限提高到 128K 后，截断率降到 0.1% 以下，training reward 继续上升，stage-specific LiveCodeBench v6 pass@1 最终达到 75.9%，相对起点提高 7.3 points。

这说明 rollout throughput、截断率和有效 batch size 必须一起看。表面上 batch 里有 8192 条 sequences，真正提供训练信号的数量可能少很多。

## 训练系统也是 recipe 的一部分

Rufus-Air 使用 GSPO、SGLang 和 Slime。Rollout 侧采用 FP8，训练侧采用 BF16，并用 sequence-level clipping、truncated importance sampling 与 Rollout Routing Replay 处理两侧差异。Agentic RL 还依赖可运行长任务的 sandbox、从 SFT 到 tool use 一致的 chat template，以及 token-in/token-out 的多轮 on-policy rollout。

对做 heterogeneous post-training 的团队，这篇 paper 最适合被改造成一份运行时 checklist：

- 每轮有多少 groups 落在 productive difficulty band
- all-fail 与 all-pass prompts 各占多少
- 多少 rollout 因截断被 loss mask
- rollout policy、训练 policy 与 checkpoint version 是否一致
- sandbox 和 verifier 的失败是否被误记为模型失败
- 每个 stage 提高目标能力时，损失了哪些已有能力

Rufus-Air 的价值在于把这些常被当成实现细节的东西放回算法本体。后训练流水线的结果，取决于模型、数据、reward、environment 和系统共同形成的闭环。

## 今天另外两篇相关论文

**Agent-Editing World Model** 不再预测高熵的 tool observations，而是识别 Critical、Exploratory 和 Noisy decisions，重写被错误假设与旧计划污染的 reasoning-action state。在线 EditAct 在六个 benchmark 上平均提高 3.2 到 6.7 points，经过 verified trajectories 做 RFT 后，收益还能蒸馏回不依赖 editor 的 policy。

**Where Does Exactly-Once Live?** 用 25,930 个故障注入 episodes 测试 agent 在 timeout、late commit 和 redelivery 下的重复副作用。当 read-back 能立即确认状态时，frontier model 的 duplicate rate 只有 0.5%；遇到仍在 flight 的请求或 transport redelivery 时，重复率升到 56% 和 74%。给所有 write 提供 idempotency key，可以把总体 duplicate rate 从 28% 降到 4%。

论文：<https://arxiv.org/abs/2609.29421>

另外两篇：<https://arxiv.org/abs/2609.28416>、<https://arxiv.org/abs/2609.29095>
