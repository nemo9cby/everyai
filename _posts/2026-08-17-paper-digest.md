---
title: "Paper Digest: 2026-08-17"
categories: [Paper Digest]
tags: [AI, Agents, Long-Horizon Research, Post-Training, On-Policy Distillation, Code Agents]
---

今天最值得看的 paper，我会选 **Beyond Final Scores: A Systematic Evaluation of Agents for Long-Horizon AI Research and Development**。

研究型 agent 的演示越来越像真正的 R&D：读代码和论文、提出方案、运行实验、观察结果，再继续修改模型或系统。评价这类 agent 时，最终分数很诱人。一个数字便于排序，也适合放进 leaderboard。

问题是，final score 几乎不告诉我们 agent 怎样走到终点。一次提升可能来自正确的问题拆解，也可能来自大量随机试验撞中局部最优。两个系统拿到相似分数，背后的失败机制可能完全不同。Agent 还会积累经验，但历史经验究竟能帮助下一步，还是会形成错误锚点，final score 同样回答不了。

这篇论文用七个 frontier models 和 36 个 long-horizon AI R&D tasks，系统研究了这些被单一分数压平的过程。

## 把研究过程拆成三个阶段

作者用 rule-based metrics 描述 agent 在一次 run 中的行为，并把研究过程拆成三部分：

- **Solution Framing**：怎样理解问题、识别约束、提出假设和规划实验
- **Execution**：能否把方案正确实现，并有效使用代码、工具与计算资源
- **Feedback Control**：怎样读取实验反馈、诊断失败、调整后续行动

这个拆分很实用。Long-horizon task 的最终失败往往具有多种来源：方向选错、实现出错、实验设计没有区分度、反馈读错，或者观察到坏结果后仍然重复原方案。只记录 reward，会把这些过程压缩成同一个零。

对 agent post-training 来说，三个阶段也提供了一套更细的 trajectory schema。我们可以标注 framing 是否合理、execution 是否忠实、feedback 是否真正改变后续 policy，然后检查哪些过程信号最能预测最终成功。

## 当前 agent 更像 engineering optimizer

七个 frontier models 已经可以提出并实现有用的工程改进。它们会组合已有技术、调参数、修代码，并在明确的反馈回路中提高目标指标。

论文同时发现，真正的方法创新依然少见。表现最好的方案大多来自已有技术的适配与组合，run-to-run variance 也很大。一次成功不足以证明系统掌握了稳定的研究能力，因为同一个 agent 再跑一遍可能走向完全不同的结果。

这个结论适合用“工程优化器”来概括。Agent 已经能在给定实验空间中做有价值的搜索，但研究问题的重新定义、新机制的提出，以及跨实验稳定积累知识，仍然很弱。

## 相同分数会掩盖不同瓶颈

论文最有价值的观察之一，是相似 final outcomes 背后可能存在不同的 process bottlenecks。

一个 agent 可能 framing 很好，执行时频繁出错。另一个 agent 能稳定写出代码，却没有根据 feedback 修正研究方向。如果只看最终 score，这两个系统都会被归到“任务失败”。它们需要的改进手段完全不同：前者需要更可靠的 execution harness，后者需要更好的实验解释与策略更新。

这也解释了为什么单纯升级 backbone model，有时无法转化成端到端收益。模型推理变强后，environment、state management、tool reliability 或 evaluation protocol 可能成为新的限制。研究 agent 必须作为一个系统来测量。

## Experience reuse 可能帮忙，也可能带偏

Self-improving agent 的核心承诺，是把过去 trajectory 变成未来能力。论文通过受控比较，检查经验在同一任务和跨任务时怎样影响后续决策。

结果并不稳定。相关经验能够缩短探索、复用有效方案，也会让 agent 过早锁定某个策略。当任务表面相似而关键机制不同，retrieved experience 很容易成为错误锚点。Memory 中的信息越丰富，agent 未必越可靠。

这个问题与普通 RAG 有明显差异。研究经验包含假设、实验条件、失败原因和适用边界。只按语义相似度检索，很容易召回一个“看起来像”的成功案例，却漏掉它成立所依赖的条件。

更好的 experience system 至少需要保存：

- 当时的问题定义与约束
- 方案依赖的假设
- 实验环境和资源预算
- 哪些反馈触发了策略修改
- 最终结论的适用范围与失败边界

Experience reuse 本身也需要 evaluation。可以测量召回后 agent 是否更快找到有效方向、是否减少重复失败，以及在不匹配任务上是否出现 negative transfer。

## Harness 会改变稳定性

作者发现 harness design 会影响 performance stability。这一点对长任务尤其重要。

短 benchmark 中，一次 tool error 可能只损失一个步骤。长达数小时的研究任务里，状态丢失、实验结果没有结构化记录、资源分配不合理或恢复机制缺失，都可能让前面的工作失效。模型能力相同，harness 仍然会改变 agent 能否持续利用已经获得的信息。

因此，long-horizon evaluation 需要同时记录 model、prompt、tools、memory、execution environment、budget 和 retry policy。缺少这些配置，单个分数很难复现，也很难定位改进来源。

## 对 post-training 的启发

这篇论文最适合继续做成 process-level training data。

可以先把 trajectory 按 Solution Framing、Execution 和 Feedback Control 分段，再为每段构造诊断标签：

- framing 是否覆盖关键约束
- 实验是否能够区分候选假设
- implementation 是否忠实执行计划
- observation 是否被正确解释
- 新证据出现后，policy 是否发生合理更新
- 引用历史经验时，是否检查了适用条件

这些标签可以用于 SFT data filtering、process reward modeling，也可以帮助 RL 做更细的 credit assignment。更重要的是，它们让失败数据重新变得有价值。一个最终没有提高 score 的 run，仍可能包含优秀的 framing 或及时的错误诊断。

下一步值得看的实验包括：哪些过程指标最能预测跨任务泛化；把高质量 framing 与 execution 分开训练是否有效；experience retrieval 加入 applicability check 能否减少 negative transfer；以及不同 harness 下训练出的 policy 能否迁移。

## 为什么今天选它

研究型 agent 最难的部分，逐渐落在稳定执行、反馈闭环与经验积累上。单一 leaderboard score 无法解释这些能力。

这篇论文给出了一套可操作的过程框架，也提供了一个克制的现状判断：frontier agents 已经是有用的 engineering optimizers，但可靠的自主研究仍受制于过程瓶颈、运行方差、经验误导和 harness 设计。

如果团队正在收集长轨迹、设计 process rewards，或者构建能够从历史实验学习的 agent，这篇论文比又一张最终分数表更值得细读。

## 另外两篇

第二篇是 **SimpleOPD: Simple Tokenizer-Agnostic On-Policy Distillation for Long-Context Reasoning**。

它研究如何把 long-context reasoning teacher 的能力迁移给 short-context students。不同 tokenizer 的 teacher 和 student 在共享文本空间中对齐，只监督覆盖相同 text span 的 tokens；student-reference KL 和 termination-token advantage masking 用来控制 response length explosion 与训练不稳定。方法覆盖 Qwen3、Qwen3.5、Intern-S2、GLM-4.7 和 Gemma-4 等不同模型家族。Intern-S2-Preview 在 ProofBench 上提升 21.2 分，达到 55.2，并在 science benchmarks 上获得迁移收益。这是一篇很容易转化为训练 ablation 的 OPD paper。

第三篇是 **Vero: Can AI Agents Build Formally Verified Software Repositories?**。

Vero 把 verified code generation 扩展到 repository level。43 个 multi-module Lean 4 repositories 来自 Python、Dafny、Verus 和 Coq 项目，覆盖 cryptographic protocols、distributed systems 等领域。Agent 需要同时保持 implementation 与 proof 跨模块一致。Benchmark 还允许 agent 正式证明 specification 不可满足，或者 reference code 本身错误。最强配置完整解决 27 个实例，在最难 repositories 上没有关闭任何 specification。它为 verifier-grounded code-agent training 提供了稀缺的、机器可检查的环境。

论文：<https://arxiv.org/abs/2608.13417>

另外两篇：<https://arxiv.org/abs/2608.14277>、<https://arxiv.org/abs/2608.13522>
