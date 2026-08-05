---
title: "Paper Digest: 2026-08-05"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, Tool Use, Post-Training, Self-Distillation]
---

今天最值得看的 paper，我会选 **TurnSight: Turn-Level Hindsight Self-Distillation for Tool-Integrated Reasoning**。

它研究的是 agentic RL 里一个很具体的难题：一条十轮 tool-use trajectory 最后拿到成功或失败的 reward，模型怎么知道是哪一轮工具选择、参数构造或中间推理真正决定了结果？

GRPO 一类方法可以在同一个问题的多条 rollout 之间比较 outcome，却很难把 trajectory-level reward 精确分配给中间 turn。Token-level self-distillation 提供更密的信号，但一个 tool interaction 内的 reasoning、tool name 和 arguments 可能得到互相冲突的 token weights。TurnSight 的答案是让未来的真实执行结果回头评价过去，并把整个 interaction turn 当作 credit assignment 的基本单位。

## 从执行结果构造 hindsight

TurnSight 先让 student 正常完成 on-policy rollout。对于第 k 个 interaction turn，它保留 student 当时能够看到的 causal context，同时构造一个 privileged context，额外加入随后一到三轮的 tool calls 和 tool results。

一个 frozen reference model 在这个 privileged context 下重新计算 student 已经生成的 token 概率。若看到执行后果后，reference 对某个 token 的 log probability 上升，说明未来证据支持这个决定；若概率下降，说明执行结果暴露了问题。

这条监督与常见的 ground-truth answer 或 reference trajectory 有重要区别。它来自 student 自己访问过的 states 和真实产生的 tool outputs，因此不会要求 student 模仿一条没有经过相同环境状态的理想路线。

论文的进一步实验也很有意思：只给 teacher 看 tool result 的效果最好，加入 ground-truth answer 反而降低表现。最终答案包含的是 trajectory-level 信息，可能掩盖某个具体 turn 的因果贡献。对于工具决策，state-aligned execution evidence 比更多的 privileged information 更有用。

## 为什么要在 turn level 聚合

一个 interaction turn 通常包含 reasoning、工具选择和参数生成。即使整体决策正确，格式 token 与参数 token 的 teacher-student gap 也可能剧烈波动。逐 token 调节 advantage 会把同一个语义动作拆成互相冲突的更新。

TurnSight 对每个 lookahead teacher 的 token gaps 取 turn-level aggregate，让一次完整工具交互共享同一条 hindsight assessment。这个选择把训练单位对齐到了 agent 真正做决定的边界。

它同时构造 lookahead depth 为 1、2、3 的三个 teacher views。短 horizon 擅长判断即时错误，例如 tool name 或 arguments 是否正确；长 horizon 可以看到某个动作在后续步骤中的延迟影响。固定使用一个 horizon 会漏掉其中一类信号。

三个 teacher 可能意见不一致。TurnSight 先对信号方向做 majority vote，再从多数方向中选择 magnitude 最大的 teacher。这样可以过滤孤立的反向判断，同时保留最强的有效证据。论文比较了 fixed lookahead、直接 fusion 和方向一致的 selection，后者在所有核心指标上都更好。

## Hindsight 只调力度，不改 RL 方向

不同问题、不同 trajectory length 和不同训练阶段会产生尺度差异很大的 teacher-student gaps。TurnSight 在同一 query 的 sibling rollouts 之间做 group-relative normalization，让某个 turn 的 hindsight evidence 表示它相对于同组其他交互有多强。

归一化后的信号经过 bounded sign-aware weighting，形成一个有限范围内的 advantage multiplier。Hindsight 与原始 RL advantage 一致时，update magnitude 增大；两者冲突时，更新被削弱。原始 advantage 的正负方向会被保留。

这个约束很关键。Execution hindsight 是局部、稠密但有噪声的证据，task reward 仍然负责定义整个 episode 的优化方向。论文在 mixing coefficient 和 modulation bound 上都观察到 unimodal trend，中等强度最好。局部 credit 太强会削弱全局目标，太弱则无法提供足够帮助。

## 实验结果

作者使用 Qwen3-4B 和 Qwen3-8B，从 base checkpoints 直接开始 post-training，没有中间 SFT。训练数据是约 2,000 个带 executable tool environments 与 programmatic feedback 的 FTRL problems。每个 query 采样 16 条 trajectories，每条最多 10 个 interaction turns，在 8 张 80GB A800 上训练 3 epochs。

评估覆盖 in-domain FTRL，以及 out-of-domain BFCL 和 ToolHop。8B 模型的整体平均分达到 42.02，前一个最强 baseline MatchTIR 是 39.03，相对提升 7.7%。4B 模型达到 37.51，MatchTIR 为 34.76。

最明显的收益出现在 BFCL Long Context 和 Miss Parameter 等需要跨多轮判断责任归属的 subsets。Qwen3-8B 的 FTRL average 从 MatchTIR 的 42.78 提升到 46.92，4B 则从 35.91 提升到 41.00。

Ablation 也支持三块设计共同起作用。在 8B FTRL 上，完整 TurnSight 的 average 是 46.92：

- 移除 turn-level aggregation：43.23
- 移除 sibling group normalization：43.65
- 用固定 teacher 取代 multi-lookahead selection：45.62

把 hindsight 覆盖到整条 trajectory 也优于只监督前五轮或后五轮。前五轮比后五轮更重要，说明早期工具决策会改变后续访问到的所有 states；完整覆盖仍然最好，因为后段 turn 也需要可靠 credit。

## 对 agentic post-training 的启发

第一，credit assignment 的粒度应该贴合 environment action 的语义边界。对于 tool agents，一整个 reasoning-call-arguments turn 比孤立 token 更自然。

第二，privileged supervision 要和 on-policy visited states 对齐。Student 自己产生的 execution outcomes 提供了天然的 hindsight source，避免引入一条状态不匹配的 gold trajectory。

第三，dense signal 适合调节 task-level advantage 的强弱。保留原始方向、限制 modulation range，可以降低 teacher noise 改写全局目标的风险。

第四，训练成本需要认真核算。每条 on-policy trajectory 都要构造三个 lookahead contexts，并运行 frozen reference scoring。它省掉了外部 judge 与人工 turn labels，同时增加了明显的 teacher inference 和 context-building 开销。对于 verl 一类 distributed rollout system，hindsight branches 的 batching、KV reuse 和异步调度会直接决定方法是否划算。

## 为什么今天选它

TurnSight 给出了一个精确又可实现的答案：让 agent 的实际执行后果成为过去决策的训练证据，再用 turn boundary、multi-horizon agreement 和 bounded advantage modulation 把这份证据变成稳定 credit。

很多 agent RL 方法知道整条 trajectory 最后是否成功。TurnSight 更进一步，试图回答每一轮为什么值得奖励或惩罚。对于 search、code、browser 和其他长链工具任务，这个问题往往决定了增加 rollout 是否真的能换来更好的 policy。

## 另外两篇

第二篇是 **PCSD: Persistent Consistency for Self-Distillation in Agentic Reinforcement Learning**。

PCSD 同样处理 on-policy self-distillation 的 teacher reliability，但它关注局部 token 信号是否持续。方法用 adaptive windows、exponential decay 和 trend-aware modulation 判断 teacher-favoring evidence 是否稳定，再将连续权重与 GRPO 联合优化。在 ALFWorld 上，它跨两个 backbones 分别超过 GRPO 15.6 和 13.3 points，并在 unseen split 上提升 15.8 points。它和 TurnSight 形成了很好的对照：一个从 token neighborhood 的 persistence 判断可靠性，一个把 execution turn 与未来 tool outcomes 作为结构化证据。

第三篇是 **PAST-Bench: Benchmarking the Foundations of Recursive Self-Improvement in Personal Agents**。

PAST-Bench 用 matched experience-on/off conditions 测量 persistent agents 是否真的因过去经验而进步。它覆盖 26 个 scenarios、204 个 episodes、七个 base models 和四种 agent frameworks，同时检查改善是否沿着预期的 save、retrieve、update pathway 发生。这个设计能区分真正的 memory reuse 与 parametric knowledge 或偶然捷径。最值得注意的是 outdated state replacement：个人 agent 的长期记忆质量取决于能否更新旧事实，持续追加记录只会把冲突留给下一次 retrieval。

论文：<https://arxiv.org/abs/2608.04007>

另外两篇：<https://arxiv.org/abs/2608.01837>、<https://arxiv.org/abs/2608.04003>
