---
title: "Paper Digest: 2026-09-18"
categories: [Paper Digest]
tags: [AI, LLM, Code Agents, SFT, GRPO, Reinforcement Learning]
---

今天最值得看的 paper，我会选 **Don't Mask the Environment: Observation Supervision Changes How Agents Explore Under RL**。

Agent trajectory 里交替出现两类 token：模型给出的 command 和 tool call，以及环境返回的 stdout、error、test result 与其他 observation。常见 SFT pipeline 只对模型 action 计算 loss，environment observation 虽然进入 context，却全部被 label mask 忽略。

ActObs 改了一件小事：同时监督 action 与 observation token。

训练数据没有增加，sequence 没有变长，参数量和 forward pass 也没有变化。实现层面只需修改 loss mask。这个微小改动在 SFT 结束时几乎看不出优势，经过相同的 GRPO 后，两条训练路线开始明显分化。

## 让模型学习 action 的后果

标准 ActionSFT 优化 agent 自己会输出的 token。ActObs 的目标额外包含环境响应：

- command 执行后出现什么 stdout
- 参数或路径错误会触发什么反馈
- test 和 build 会返回怎样的结果
- 当前 action 会把 terminal state 推向哪里

部署时，agent 依然不会生成 environment observation。Observation loss 的作用发生在 shared representation 中，它迫使模型保留 action 与 consequence 之间的结构。

论文使用同一份 50,000 条 multi-turn terminal trajectories，共 0.71B tokens，其中 observation 约占 45%。Qwen3-4B 与 Qwen3-8B 都只训练一个 epoch。随后，所有 checkpoint 接受同样的 GRPO：2,392 个 containerized tasks、135 steps、每题 16 条 rollouts，reward 来自 binary verifier。

对照组包括标准 ActionSFT，以及先做一轮 observation-only SFT、再做一轮 action-only SFT 的 Obs→Act。后者接触了同样数量的 observation supervision，却没有得到 ActObs 的收益。这说明 joint learning 的训练几何很重要，单独增加一个 world-model warm-up stage 无法复现结果。

## SFT 分数接近，RL 结果明显不同

主要评测是 Terminal-Bench 2.0，共 89 个 OOD terminal tasks。SFT 后，4B 模型的 ActionSFT 与 ActObs 很接近：pass@1 分别为 4.5% 和 4.4%，pass@8 都是 14.0%。

经过 GRPO 后，ActObs 在每个 sampling budget 上领先：

- pass@1：7.2% 对 5.6%
- pass@4：12.5% 对 11.1%
- pass@8：15.5% 对 14.0%
- pass@16：19.1% 对 18.0%

4B 的 pass@1 相对提升达到 29%。

8B 呈现另一种 trade-off。ActionSFT 在 pass@1 上更高，12.3% 对 11.0%；ActObs 随 sampling budget 增长逐步反超，pass@8 达到 23.6% 对 21.0%，pass@16 达到 27.0% 对 23.6%。它总共解决 24 个任务，ActionSFT 解决 21 个，其中三个任务此前没有任何 SFT policy 或 action-only GRPO policy 解出。

这个现象很适合用 exploration support 理解。ActObs 对单次成功率的影响取决于模型规模和 loss weight，它更稳定的优势出现在 repeated sampling 与 distinct task coverage 上。

## Cross-domain code editing 的提升更大

论文还测试了 aider-polyglot 的 225 个 multilingual code-editing tasks。这些任务没有出现在 SFT corpus 或 RL environments 中，任务形态也与 terminal benchmark 不同。

4B ActObs checkpoint 在 SFT 后其实落后于其他方法。GRPO 改变了结果：

- pass@1 达到 13.9%，ActionSFT 为 9.7%，提升 4.2 points
- pass@4 达到 25.3%，ActionSFT 为 20.4%，提升 4.9 points

相对增幅分别是 43% 和 24%。8B 的 ActObs 也在 plain-GRPO policies 中取得最好的 pass@1 与 pass@4。

这组结果很关键。ActObs 没有直接教更多 code-editing examples，收益来自更适合后续 RL 的 initialization。SFT checkpoint 的即时分数不足以判断它是否给 RL 留出了好的优化空间。

## 梯度很快变得近乎正交

作者在相同 held-out trajectories 上分别计算 action-token gradient 与 observation-token gradient。

训练初期，两者还有一定程度的对齐。随着 SFT 进行，observation gradient 与 action gradient 很快接近正交。Action-only update 因而无法覆盖大部分 observation learning signal，并且 ActionSFT 对 environment feedback 的预测能力会降到 base model 以下。

ActObs 持续优化这部分正交分量。它在 command output、shell prompt、error line 和其他 payload text 上都保留了更好的 next-observation prediction，并没有只记住固定 response header。

这个分析把结果连成了一条完整链路：

1. Loss mask 决定 SFT 保留哪些 trajectory structure。
2. Action-only SFT 对 demonstration action 发生单侧 specialization。
3. Joint supervision 保留 environment consequence prediction。
4. GRPO 可以用更小的 policy movement 找到新成功路径。
5. 最终 policy 保留更多有用 entropy，在 repeated sampling 下覆盖更多任务。

## 更多 entropy 只解释了一部分

ActObs 在 GRPO 后具有更高的 self-entropy，尤其体现在 command 后半段的 arguments、flags 和 paths。这些位置往往决定一个 terminal action 是否真正可执行。

作者做了两个重要控制。第一，把 ActionSFT policy 的 sampling temperature 调高到与 ActObs 相同的 entropy，pass@k gap 依然存在。第二，ECHO 在 RL 阶段继续加入 next-observation loss，最终 entropy 更高，却没有全面超过 ActObs。

所以 entropy 本身不构成充分条件。ActObs 的组合更特殊：它保留较高 entropy，同时 endpoint KL 更小，最终 policy 离自己的 SFT initialization 更近。GRPO 无需大幅重排 probability mass，就能发现更多有效 command variants。

## 对 post-training pipeline 的直接意义

这篇论文给出的 intervention 非常便宜，值得成为 agent SFT 的标准 ablation。

训练日志至少应该增加几项指标：

- action loss 与 observation loss
- 两类 gradient 的 cosine similarity 和相对 norm
- SFT initialization 与 RL endpoint 的 KL
- command-token position 上的 entropy
- pass@1、pass@k 与 distinct tasks solved
- next-observation prediction 在 stdout、error、prompt 和 payload 上的分项表现

数据清洗也会变得更重要。真实 agent traces 可能含有时间戳、随机 ID、超长日志、重复 dependency output、敏感字段和其他低价值 token。把所有 observation 无差别纳入 loss 可能浪费 capacity，甚至鼓励模型记忆噪声。合理的下一步包括 observation filtering、content-aware weighting，以及只监督与 action consequence 高度相关的片段。

论文目前只验证了 Qwen3-4B、Qwen3-8B、terminal tasks 和 aider-polyglot。更大的模型、GUI agents、web agents、稀疏 reward 的 repository repair，以及含有图片或结构化 tool response 的轨迹都需要独立确认。8B 上出现的 pass@1 与 pass@k trade-off 也说明，observation-loss weight 应该和产品的 sampling budget 一起调优。

即便有这些边界，ActObs 仍然是今天最值得复现的结果。它提醒我们，agent training data 的价值不只存在于模型说了什么，也存在于世界随后发生了什么。

## 另外两篇

第二篇是 **DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression**。

它面向 input-heavy 的长时 agent workloads，使用 552B multimodal MoE，decode 每个 token 激活 16B 参数，prefill 只激活 8B。Compressed Sparse Attention 2 的 cross-layer KV reuse 配合 FP4 KV cache，把常驻 HBM 的 global KV footprint 压到每 token 890 bytes，约为 DeepSeek-V4-Flash 的四分之一。SWA Bounded Replay 进一步把 SSD 或 host memory 中的 persistent KV footprint 压到原来的约八分之一。

第三篇是 **An Empirical Study of Harness Design for Coding Agents**。

作者在 SWE-Bench Verified 与 Terminal-Bench 2.1 上测试 176 个 matched settings，分别改变 planning、action space、context strategy 和 context budget。Rule-based elision 后接 LLM summarization 的整体效率最好；recoverable elision 增加了系统复杂度，模型却很少主动取回内容。Planning 对弱模型主要提供 accuracy scaffold，对强模型更像 cost saver。Bash 能力强的模型只用 bash interface 也能取得良好表现，并显著降低成本。

论文：<https://arxiv.org/abs/2609.20715>

另外两篇：<https://arxiv.org/abs/2609.19969>、<https://arxiv.org/abs/2609.20804>
