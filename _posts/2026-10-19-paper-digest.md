---
title: "Paper Digest: 2026-10-19"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Reinforcement Learning, Post-training, SWE]
---

今天最值得看的 paper 是 **FC-SWE: Failure-Conditioned RL for Long-Horizon Software Engineering Agents**。

它抓住了 code agent 训练里一类经常被浪费的数据：失败后的恢复过程。

一个 agent 提交 patch，tests 失败，verifier 已经给出了非常具体的证据。常规 GRPO 通常把这一轮当成一个 reward 为零的完整 trajectory，下一条 rollout 再从原始 issue 独立采样。失败 patch、测试输出与后续修复之间的因果关系没有进入训练。

FC-SWE 把这段关系保留下来。第一次 patch 验证失败后，环境把 repository 恢复到初始状态，再把原始 issue、失败 patch 和 verifier feedback 一起交给 agent，生成 recovery trajectory。

在全部 500 个 SWE-bench Verified tasks 上，Qwen3.5-4B + SWE-agent 达到 41.7% Resolved@1 和 52.8% Resolved@2。相同设置下，GRPO 是 38.9% 和 48.5%。训练时每条 chain 最多只有两次尝试，但测试时给到 11 次 verifier-guided attempts，累计解决率达到 70.7%。

## 把失败后的第二次尝试纳入 RL，并没有看起来那么直接

Recovery trajectory 是否存在，取决于第一次尝试有没有失败。第二次尝试的输入也由第一次生成的 patch 和 verifier output 决定。因此，一组 rollout 的成员和上下文都受 policy outcome 影响。

如果直接把一条 chain 的最终结果分给所有 attempts，会产生错误 credit。假设第一次失败、第二次修好，reward sequence 应该是 `[0, 1]`。把最终 success 回填成 `[1, 1]`，等于奖励了制造问题的第一条 trajectory。

FC-SWE 使用 **trajectory-local reward**：每次提交的 patch 只接收自己的 verifier outcome。后续 recovery 成功，也不会改写前一次失败的 reward。

第二个问题是 comparison group。每条 initial trajectory 一定存在，recovery trajectory 只在前一次失败后产生。把未执行的 recovery slots 填进 GRPO group 会扭曲均值和 advantage。

论文提出 **active-set advantage estimation**，只汇总同一个 issue 下真正执行过的 initial 和 recovery trajectories。失败样本仍留在 group 中，为成功 recovery 提供对照；从未生成的 attempts 不参与统计。

这两个设计很小，却决定了训练信号是否忠实反映 agent 的行为。

## Failure context 需要训练，临时塞进 prompt 不够

论文里一个很实用的发现是，原始 Qwen3.5-4B 和 GRPO model 在测试时拿到 failure context，recovery 表现反而低于完全独立的 resampling。模型看到一份失败 patch 和一串 test logs，并不天然知道怎样利用它们，甚至会被上一轮错误路径锚定。

FC-SWE 经过 recovery training 后，feedback-conditioned retry 才超过 independent resampling。Ablation 也支持这一点：去掉 failed patch、verifier feedback 和 prior interaction history，只保留一次 fresh retry，Resolved@2 从 52.8% 降到 47.9%，conditional recovery 从 19.0% 降到更低水平。只给 patch 有帮助，再加入 verifier evidence，recovery 还能增加 3.3 percentage points。

这意味着 verifier 的价值可以超过一个 binary reward。Failing tests、stack trace、lint errors 与 grader diagnostics 都能成为下一次 action 的 observation，但 policy 必须在训练中学会读取这种 evidence。

## 对 code-agent post-training 的启发

FC-SWE 给出了一种很自然的数据组织方式：训练单位可以是一条由 verifier outcome 驱动的 attempt chain。

每个 attempt 仍是完整的 tool-use trajectory，repository 在 attempts 之间 clean reset，失败 patch 和 diagnostics 通过显式 context 传递。这样既保留真实 recovery 过程，又避免 workspace state 悄悄泄漏到下一次尝试。

这套方法也很适合扩展到更丰富的 verifier：

1. 把 raw test output、结构化 failing-test summary 和 learned diagnosis 分开比较。
2. 让 recovery chain 在不同 failure types 上使用不同 attempt budget。
3. 研究训练于一种 agent harness 的 recovery policy，能否迁移到另一种 tool protocol。
4. 在分布式 rollout 系统里，让失败事件动态触发后续 workers，同时维护 per-attempt lineage 和 active comparison set。

对 Nemo 的方向，这篇 paper 把 SFT/RL、code agents、verifier feedback 与异步 rollout orchestration 放进了同一个问题里。它的价值不只在 SWE-bench 多出的几个 points，也在于明确了 recovery data 应该怎样进入 policy optimization。

## 今天另外两篇相关论文

**Tree-Aware Posterior Distillation for Long-Horizon Agentic Reinforcement Learning** 从共享 interaction prefix 分支多条 continuations，把 terminal rewards 递归回传成 turn-level credit，再构造一个相对 frozen behavior policy 的有界 teacher distribution。它不需要 value network、process labels 或外部 teacher，并让 Qwen3-30B-A3B 在 Deep Research 上比 GRPO 高 4.97 points。

**Skip the Compressor: Coding Agents Fix More Issues When They Pick Whole Files** 给出一个非常实用的结论：在相同 token budget 下，挑选少量完整文件，比 chunk retrieval 和 prompt compression 更有利于修复真实 issue。GPT-5.6 和 Claude Opus 5 在 SWE-rebench 上分别达到 43.7% 和 46.7%，说明 code agent 既需要找到正确文件，也需要看到未经删改的 exact code。

论文：<https://arxiv.org/abs/2610.07898>

另外两篇：<https://arxiv.org/abs/2610.23181>、<https://arxiv.org/abs/2610.21073>
