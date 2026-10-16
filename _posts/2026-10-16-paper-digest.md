---
title: "Paper Digest: 2026-10-16"
categories: [Paper Digest]
tags: [AI, LLM, Agents, Reinforcement Learning, Post-training, AutoResearch]
---

今天最值得看的 paper 是 **Athanor: An Agent-Native System for Autonomous Post-Training Research**。

论文做了一场很直接的实验：给四个 Claude Opus 5.5 agents 各一台 8-GPU node，让它们从 Qwen3-4B-Base 出发，自己找数据、写训练代码、跑 SFT 和 RL、分析 evaluation、交换 checkpoint，并决定下一轮实验。人类负责预先设定规则、维护机器和最后评分，不提供研究建议。Agents 也不能调用外部 teacher model。

104 小时后，它们交出的模型在六项 development tasks 上得到 67.3，sealed held-out suite 上得到 48.2。相同 evaluator 下，Qwen3-4B-Instruct-2507 分别是 66.5 和 47.9。

这个结果最值得关注的部分，是让研究 loop 持续运转的系统设计。

## 把 post-training 研究变成一组可编程操作

Athanor 给 agent 暴露的核心动作很少：提交 training job、请求 evaluation、导入数据、保存和提交 artifact。Agent 保留研究判断权，可以检查数据和模型输出、修改 starter code、写新算法、组合 checkpoint。Environment 负责 GPU allocation、job execution、evaluation、checkpoint storage、失败恢复和结果通知。

这套分工解决了 autonomous ML research 里最消耗时间的一层摩擦。一次 GRPO run 可能需要数小时，一次完整 evaluation 也需要几十分钟。Agent 如果同步等待，每轮只能串行提出一个想法，大量 GPU 和 context 都会闲置。Athanor 允许多个实验并行在后台运行，job 脱离单个 agent session 生存，结果完成后再推送回来。

论文把这套设计总结为三个原则：

1. Research decision 与 execution 分离。Agent 决定做什么，environment 负责可靠地执行和测量。
2. Research state 外部化。每次请求、代码、数据、checkpoint lineage、分数、transcript 和 note 都进入 durable record。
3. 保持 loop productive。训练和评测异步进行，agent 可以同时维护多个 in-flight experiments。

第二点尤其重要。长时间 research campaign 一定会跨越 session、context window 和机器故障。Athanor 把证据链放在 agent context 外面，因此 session 可以重启，agent 可以换 node，团队成员也能读取彼此的结果和 checkpoint。最终每个分数都能追溯到 base model、训练程序、输入数据和所有 parent checkpoints。

## 四个 agents 最后找到了什么 recipe

最终模型来自四段大家都熟悉的方法，价值落在组合顺序和局部修正上。

第一阶段是 SFT。四个 agents 在开始一小时内都选择了同一份 public reasoning corpus 作为主要数据源，再加入覆盖六种能力的数据和 ALFWorld trajectories。最好的 development score 从 16.1 提高到 49.7。

第二阶段是 GRPO。一个 agent 发现，SFT model 用 greedy decoding 时容易陷入 repetition loop，而常规 sampled rollouts 很少出现这种失败，RL 因而学不到修复信号。它修改 rollout logic，让四分之一轨迹使用 greedy decoding。重复输出会得到零 reward，这个小改动随后被其他 agents 复用，最好成绩提高到 61.7。

第三阶段是跨 agent weight merging。Agents 平均多个队友的模型，再把合并模型相对 base model 的 task vector 外推 1.2 到 1.3 倍，成绩达到 65.5。

第四阶段是 self-distillation。系统对每个问题采样四个回答，保留 rule-based verifier 判定正确且最短的一个，再做 rejection-sampling fine-tuning。两个 agents 分别完成 self-distillation，最后平均它们的模型，得到团队 submission。

完整 campaign 留下 500 个 jobs、255 次 full evaluations、174 个经过完整评测的 checkpoints，以及 57 次 agents 之间的 checkpoint fetch。Hour 21 以后，每个新的 best checkpoint 都结合了多个 agents 的训练结果。这里的 multi-agent gain 很具体：想法、代码和 weights 在几小时内传播，最终模型拥有清晰的跨 agent lineage。

## 这场实验也暴露了 agents 的研究边界

四个 agents 使用相同 foundation model，先验高度相似。它们在第一小时就收敛到同一份 public corpus，后续只尝试了少量替代路线。论文认为，当 execution friction 被大幅降低后，最明显的限制变成了探索广度。

Agents 对数据的 format、length、difficulty 和 scorability 很敏感，却很少系统检查答案本身是否正确。最终 recipe 里 76% 到 87% 的 SFT rows 来自同一份 distilled corpus。官方 Qwen3 Instruct 的平均分虽然被追平，能力分布仍有明显差异：团队模型在 multi-step action 和 code 上更强，在 mathematics 与 logic 上更弱。均值掩盖了一部分 capability imbalance。

成本也值得冷静看待。整场 campaign 约花费 11,800 美元，其中 GPU 约 10,800 美元，agent inference 约 1,074 美元。Agents 已经能完成大量 post-training engineering 和局部 research，但当前配置依然依赖 32 张 GPU、精心搭好的受控环境、公开的 distilled data，以及明确可计算的 benchmark rewards。

## 对 autonomous AI research 最有用的结论

Athanor 展示了一种可验证的 autoresearch 形态：研究者拥有足够自由度，执行环境提供有限但可靠的 primitives，evaluator 被隔离，held-out suite 保持密封，所有实验进入持久化 lineage。

这比给 agent 增加更多角色描述或更长 prompt 更接近真正的基础设施问题。训练系统需要让 agent 快速启动实验、并行等待结果、恢复失败、共享模型，同时阻止它修改 evaluator 或接触 held-out data。研究产出也需要包含完整 provenance，才能区分真正的能力增益、evaluation noise 和 benchmark overfitting。

一个很适合复现的缩小版，可以只用两个 agents、一到两台机器和 1B 到 4B base model。重点测量三件事：异步 scheduling 能提高多少有效实验数，shared checkpoint lineage 是否带来可重复的团队增益，sealed evaluation 能否抵抗 agents 在长时间搜索中的隐性 overfitting。

## 今天另外两篇相关论文

**Waypoint-OPD: Transferring RL-Induced Policy Shifts via Gated On-Policy Distillation** 利用 teacher RL trajectory 中的 intermediate checkpoint，比较 early 与 late policy shift 的方向是否一致。两段变化冲突时，系统压低 endpoint distillation signal。它只增加一次 frozen-teacher scoring pass，并在 Qwen3 系列 students 的数学和代码任务上取得六组设置中的最佳 domain-average result。

**Cowork-Gym: Synthesizing Interactive Work Environments at Scale** 用 build-execute-revise loop 生成 64.9K 个 interactive work environments。Stateful user simulator 可以在任务中途对话并直接修改 workspace。作者用这些环境对 9B、27B 和 35B-A3B Qwen models 做 SFT、RL 与 on-policy distillation，在六个 agent benchmarks 上分别带来 8.7、6.4 和 4.1 points 的平均提升。

论文：<https://arxiv.org/abs/2610.21055>

另外两篇：<https://arxiv.org/abs/2610.21601>、<https://arxiv.org/abs/2610.17670>
