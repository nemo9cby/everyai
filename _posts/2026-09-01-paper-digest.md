---
title: "Paper Digest: 2026-09-01"
categories: [Paper Digest]
tags: [AI, Post-Training, RLVR, Distillation, LoRA, Tool Agents]
---

今天最值得看的 paper，我会选 **Does On-Policy Distillation Really Distill? From Noisy Teacher to Self-Improvement**。

On-policy distillation（OPD）看起来像一种更密集的 RLVR。Student 先采样 trajectory，teacher 再给每个 token 提供 advantage。相比只有最终成败的 sparse reward，这种 token-level supervision 理应更容易训练。

这篇论文追问了一个很基础的问题：student 的提升究竟来自 teacher 知识，还是来自 update rule 本身？

作者的答案相当尖锐。Teacher signal 含有大量噪声，而且 teacher 越大，噪声比例反而越高。保留或删除这些 noisy supervision，student 最后会收敛到相近的性能。把 teacher advantage 全部替换成一个固定的负数，也能达到相近结果。

## OPD 在学习什么

OPD 的 teacher 需要给 student 生成的 trajectory 打分。这些 token 来自 student policy，对 teacher 属于 off-policy sample。Teacher 对这类 trajectory 的 token-level judgment 并不天然可靠。

论文进一步检查 gradient 集中在哪里，发现更新主要发生在 student 自己认为概率较低的 token 上。固定负 advantage 的效果也说明，训练的关键机制更接近压低 tail token，而非精确模仿 teacher 的相对偏好。

这个结论会改变对 OPD 成本的判断。一个标准 pipeline 需要同时服务 student rollout model 和 teacher scoring model，保存 token-level score，并处理两个 tokenizer、checkpoint 与 serving stack 之间的对齐。若 teacher signal 的主要价值可以由 student uncertainty 代替，整个 teacher stage 都有机会被移除。

## OPSA：用 entropy 代替 teacher

基于上述观察，作者提出 On-Policy Self-Adaptation（OPSA）。

OPSA 不调用 teacher。它根据每个位置的 entropy 生成自适应负 advantage：高 entropy 位置得到更强的学习信号，tail token 被压低，概率质量在 head token 之间重新分配。

在 Qwen3-1.7B 上，OPSA 相对 base model 将 AIME24 Avg@32 提高 35.41 分，对应 263% relative gain。它还比 OPD 高 16.77 分，并在三个 benchmark 上把 Pass@32 提升到 base model 的两倍以上。

这些数字很亮眼，更有价值的部分是消融链条：teacher supervision 有噪声，去掉噪声没有明显改善，固定负 advantage 可以匹配 teacher advantage，最终 teacher-free 的 entropy rule 又能超过 OPD。每一步都在缩小真正有效机制的范围。

## 对 post-training pipeline 的意义

Teacher-free on-policy training 会带来很直接的 systems benefit。

Rollout worker 只需输出 token、log-probability 和 entropy。训练侧可以在本地构造 advantage，无需等待另一套大模型完成 token scoring。数据 schema、模型版本组合和 serving capacity 也会简单很多。

它同时留下一个重要边界。Math reasoning 中的低概率 token 往往与错误分支相关，但 code 和 tool agent 里，罕见动作有时正是探索到正确解所必需。直接压低 tail 可能减少错误，也可能损失有价值的 exploration。OPSA 是否能迁移到 code RL、long-horizon tool use 和开放式任务，需要单独验证。

最值得做的 follow-up 很具体：在同一 base model、同一 rollout budget 下，对比 RLVR、OPD、固定负 advantage 和 OPSA，分别测试 math、code 与 agent task，并追踪 entropy、pass@k、trajectory diversity 和 rare-correct-action recall。

## 另外两篇

第二篇是 **Normalized Low-Rank Adaptation**。

NoRA 从 LoRA 的初始化动态出发。由于 up-projection 初始化为零，早期优化主要由 down-projection 决定。论文对 down-projection 做 normalization，发现持续 normalization 或只在初始化时做一次，都能改善收敛、稳定性与 catastrophic forgetting。它覆盖 pretraining、SFT 和 RL，不增加 trainable parameters，也没有 inference-time cost。对于已经大量使用 LoRA 的训练栈，这是一个很便宜的 reproduction target。

第三篇是 **CAST: Critique-Aware Supervision for Training Reliable Long-Horizon Tool-Calling Agents**。

CAST 把 sparse outcome 转成 action-level critique supervision。系统分析长 trajectory，在 partial observability 下生成结构化 action-validity rationale，训练 critique model，再用 critique-aware data 优化 policy。Qwen3-family 模型在 dynamic tool-use benchmark 上提高 repeated-trial reliability，Retail 的 pass^4 超过 GPT-OSS-120B 十个百分点以上，并在 out-of-domain Telehealth 上再提高 9%。它把“偶尔成功”与“多次运行都可靠”明确区分开来。

论文：<https://arxiv.org/abs/2608.31046>

另外两篇：<https://arxiv.org/abs/2608.31036>、<https://arxiv.org/abs/2608.30147>
