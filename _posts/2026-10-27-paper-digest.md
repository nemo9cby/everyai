---
title: "Paper Digest: 2026-10-27"
categories: [Paper Digest]
tags: [AI, LLM, Tool Agents, SFT, Post-training, Agent Systems]
---

今天最值得看的 paper 是 **Trajectory2Gym: Decision-Grounded Compositional Interaction Synthesis for Tool-Use Agents**。

Tool-use agent 的训练数据通常从成功轨迹中筛选：任务完成了，API 调通了，这条 trajectory 就进入 SFT。Trajectory2Gym 指出，这个标准漏掉了两个决定 agent 能否真正学会工作的条件。

第一，训练数据要覆盖多步依赖。后面的 action 往往取决于前面的 tool result、用户补充信息、环境状态变化或一次失败后的 recovery。第二，每个被监督的 action 都必须能由 agent 当时可见的 history 推导出来。只验证最终成功，无法排除 teacher 使用隐藏 identifier、过期请求或 learner 看不到的信息。

## 成功轨迹也可能教错东西

论文给出的例子很直观。用户先要求更换水瓶和台灯，随后又改口，只确认更换台灯。一个额外更换水瓶的 API call 仍然可以成功执行，但它已经违背了用户最新请求。

Outcome filtering 会保留这次调用，因为工具返回成功。Trajectory2Gym 会检查 decision-time visible history，只监督有 catalog 证据且符合最新确认的台灯操作。

作者还发现，34,772 条已经通过结果验证的候选轨迹中，40.9% 最多只有两个 supervised decisions，只有 10.0% 包含一次真实错误之后的合格决策。数据量看起来很大，information acquisition、cross-step dependency 和 recovery 等关键结构的 support 仍然很薄。

## 三段式训练数据管线

Trajectory2Gym 把数据构造拆成三个相互依赖的阶段。

1. **Compositional interaction synthesis**：先找出 support 不足的 capability paths，再围绕这些路径生成任务和初始状态，让 teacher 在原生环境里执行完整 interaction。
2. **Grounded supervision**：保留 learner 能看到的完整交互作为 context，同时给每个 assistant decision 加 supervision mask。只有能被 visible evidence、execution 和 workflow order 共同支持的 action 才成为 target。错误 action 可以留在 history 中作为 recovery context，但不会被模仿。
3. **Structural-support learning**：根据 learner scale 选择每类结构需要多少 support，再用 parent-normalized SFT 训练。

最后一个设计很容易被忽略。普通 token-level SFT 会让 decision 更多、序列更长的 interaction 获得更大 loss weight。Parent-normalized weighting 把完整 semantic interaction 作为训练权重单位，避免长轨迹仅凭长度主导更新。

## 小模型超过大模型初始化

作者在 Qwen3 1.7B、4B、8B、14B 和 30B-A3B 五个规模上测试，Trajectory2Gym 在 Tau1、Tau2 和 BFCL 上全部带来提升，三个 benchmark 的等权平均提高 **16.9 到 29.8 个百分点**。

更醒目的结果出现在 Tau2：post-trained 4B 达到 **69.17**，超过 public 30B-A3B initialization 的 **28.33**。在这类 stateful、multi-turn tool use 中，训练数据是否真正呈现依赖关系，比单纯扩大参数量更能改变行为。

## 对 agent post-training 的启发

这篇论文把 trajectory quality 变成了几个可以落实的工程检查。

- 每个 target action 都要通过 policy-visible prefix 审计。
- 训练集应统计 capability path 的 support，而只统计成功任务数会掩盖结构缺口。
- Authentic errors 可以作为 recovery context，错误 decision 本身应从监督目标中移除。
- 需要区分 outcome correctness、execution validity 和 decision grounding。
- 长轨迹要控制训练权重，避免 action count 隐式改变采样分布。

对于 SFT/RL pipeline，一个自然的后续实验是固定 rollout pool，对比 outcome-filtered SFT、decision-grounded SFT 和 verifier-guided RL。这样可以进一步分解收益来自更广的 compositional coverage，还是来自更干净的 step-level credit assignment。

## 今天另外两篇相关论文

**Eat Your Precision: Precision Objective Mitigates Reward Hacking in Rubric-based RL** 研究 rubric reward 的 recall bias。模型为了覆盖更多 rubric 会写得更长，也加入更多无必要或错误内容。FORK reward 同时优化 recall 与 response precision，要求每个 response unit 都有必要性并通过 factual-correctness grader。相比原始 recall reward，论文报告对 base policy 的 win rate 在 RubricHub-Medical 从 10.6% 提升到 68.0%，在 HealthBench 从 22.1% 提升到 74.1%。

**Trial by Trace: Benchmarking AI Agent Harnesses for Reliability and Failure Attribution** 把 model performance 和 harness reliability 分开评估。它覆盖 context、execution、governance、lifecycle、observability、tools 和 verification 七类机制。4 个 harness、9 个 models、3,420 次运行中，有 31.1% 的 runs 虽然完成任务，仍至少违反一项 harness reliability criterion。

论文：<https://arxiv.org/abs/2610.33568>

另外两篇：<https://arxiv.org/abs/2610.33512>、<https://arxiv.org/abs/2610.32285>
