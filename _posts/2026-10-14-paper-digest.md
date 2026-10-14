---
title: "Paper Digest: 2026-10-14"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Reflection, Reinforcement Learning, Post-training]
---

今天最值得看的 paper 是 **Agent-Reflex: Teaching Language Agents to Act Reflectively**。

真实环境里的 agent 很少收到干净、确定的反馈。MCP tool 可能返回噪声，网页状态可能发生变化，用户也会在中途指出错误。一个 action 只要走错一步，后面的 reasoning 很容易建立在错误观察之上，最终形成一条看似完整、实际已经偏航的 trajectory。

Agent-Reflex 想训练出一种更具体的能力：agent 能否主动发现交互中的异常，反思刚才的判断，并从失败中恢复。

作者在 Qwen3-8B 上构建了一个三阶段 post-training pipeline。最终模型在五个 long-horizon tool-use benchmarks 上达到 58.6 overall score，在专门测试噪声鲁棒性的 AgentNoiseBench 上达到 29.1，超过论文中的 environment-scaling 和 reflection baselines。

## 第一阶段：从真实失败里发现 reflection skill

Agent-Reflex 的第一步叫 **Environment Reflection Skill Discovery（RSD）**。

系统会向 tool feedback 注入与环境相关的 perturbation，再让一组能力不同的模型执行任务。这样可以获得同一类情境下的成功与失败 trajectory。接下来，pipeline 匹配这些 success-failure pairs，从差异中总结可复用的 reflection meta-skills。

这里最有价值的设计，是让 reflection skill 落在具体交互证据上。Agent 需要识别的是某种可观察的故障模式，例如 tool response 与预期状态不一致、当前 action 没有产生进展、用户纠正暴露了此前被忽略的约束。抽象出来的 skill 仍然保留了它来自哪类 failure 的数据来源。

对于 agent data pipeline，这提供了一个很直接的 flywheel：

1. 对 tool feedback 进行可控扰动。
2. 收集多种 policy 的成功与失败 rollout。
3. 在相似任务和状态之间匹配 trajectory。
4. 从行为差异中提取 recovery pattern。
5. 把新的 failure mode 放回下一轮数据生成。

## 第二阶段：把 recovery 写进 SFT trajectory

第二阶段是 **Reflective Supervised Fine-Tuning（ReSFT）**。

RSD 得到的 skills 被转换成三类监督：self-reflection、skill recovery 和 user-centric correction。模型学习的不只是生成一段「重新考虑」的文字，还要根据反思结果改变接下来的 action，并在用户指出问题后修正自己的计划。

这一步解决了常见 reflection prompt 的弱点。单纯要求模型「检查你的答案」经常带来更长的输出，却没有稳定改变执行策略。ReSFT 用成功恢复轨迹告诉模型：什么信号值得停下来检查，应该回退到哪个决策点，恢复后该采取什么行为。

它也适合与现有的 agent SFT pipeline 结合。数据团队可以保留完整的 failed rollout，把错误对应到 tool observation、state transition 和用户 correction，再生成带有明确恢复动作的训练样本。相比只保留最终成功轨迹，这类数据包含了模型如何从错误状态回到可行路径。

## 第三阶段：在 noisy interaction 中做 on-policy RL

最后一阶段是 **On-Policy Reflective Policy Optimization（ORPO）**。

它把 noisy environment 中的 agentic RL 与 reflection-guided on-policy distillation 结合起来，继续强化主动反思和失败恢复。On-policy 很关键，因为模型经过 ReSFT 后会访问新的状态分布。训练需要观察当前 policy 自己制造的错误和恢复机会，静态数据很难覆盖这些分支。

从工程角度看，这个阶段也最昂贵。长程 rollout、tool execution、环境 perturbation、trajectory pairing 和 reflection judging 都需要保持数据 lineage。每个恢复动作应当能追溯到触发它的 observation，以及它是否真正提高了后续成功概率。否则训练很容易奖励表面上的自我批评，模型学会频繁反思，却没有获得更可靠的行为。

## 真正值得验证的三个问题

Agent-Reflex 给出了完整 recipe，下一步需要更严格地拆解收益来源。

第一，matched success-failure pair 带来了多少增益？它可能比显式 reflection label 更重要，因为模型看到了相似状态下两条 trajectory 的分叉点。

第二，训练中的 noise 是否接近生产环境？随机删除或替换 tool feedback 很容易规模化，真实故障还包括 stale state、部分成功、权限错误、格式漂移和用户目标变化。Perturbation distribution 决定模型学到的是通用 recovery，还是一组可识别的合成模板。

第三，主动反思的成本如何控制？Reflection 过少会错过恢复窗口，过多会增加 latency，并让简单任务陷入循环检查。一个实用 agent 需要同时优化 task success、recovery rate、tool cost 和额外 steps。

对做 SFT、RL 和 agent infrastructure 的团队来说，这篇论文最有启发的地方，是把 failure recovery 变成一条可以持续生产的数据管线。工具噪声和用户纠正本来是部署中的麻烦，也可以成为训练下一代 agent 的高价值监督。

## 今天另外两篇相关论文

**Tev: Scaling Long-Horizon Agentic Environments with Dense Progress Rewards** 构建了包含 1,170 个 verified environments 的 Tev-Gym。每个任务先生成一份隐藏的 latent-world specification，描述完整 workflow 和每一步的 expected state，再从中渲染 agent-facing environment。这个 hidden specification 同时提供 progress metric，论文用 progress-potential reward shaping 把稀疏 terminal reward 转换成 dense per-step reward。它把环境生成和 reward design 放进同一个结构化表示，适合研究长程 agent RL 的训练效率和 reward hacking 风险。

**TPPO: Tracking-Value PPO For Language Model Reasoning** 关注 sparse terminal reward 下的 token-level credit assignment。它把相邻 prefix value 的变化表示为 log-odds space 中的 additive update，再累计 position-wise scalar outputs 重建 value logits。论文报告，相比 PPO，TPPO 在 Qwen3-4B-Base 和 Qwen3-8B-Base 的 aggregate math reasoning scores 上分别提升 7.0 和 8.4 分。这条路线值得在 code reasoning 上验证，尤其要观察 prefix value 能否帮助定位 trajectory 从哪里开始失去成功概率。

论文：<https://arxiv.org/abs/2610.17953>

另外两篇：<https://arxiv.org/abs/2610.18868>、<https://arxiv.org/abs/2610.16794>
