---
title: "Paper Digest: 2026-09-16"
categories: [Paper Digest]
tags: [AI, LLM, Code Generation, Code Agents, RLVR, GRPO]
---

今天最值得看的 paper，我会选 **ExecuCritic: Calibrated Critic Shaping for Code Generation with Verifiable Rewards**。

代码模型做 RLVR 时，unit tests 可以给出客观的 pass/fail verdict，但整个程序经常只得到一个 bit 的奖励。程序究竟错在 API、类型、边界条件、复杂度还是核心算法，这个信号不会告诉 policy。Credit assignment 因此很难，尤其是长程序和 repository repair。

ExecuCritic 的做法是让 coder 与 critic 使用同一批 execution rollouts 联合训练。Critic 预测程序通过概率，也输出简短的 failure diagnosis。Coder 仍然以 executor reward 为锚，只在 critic 的排序与真实执行结果保持正相关时，才把 critic score 加入 GRPO advantage。

这条 calibration gate 是整篇论文的关键。Critic 可以提供更密集的训练信号，同时没有获得无条件影响 policy 的权力。

## 一个受 executor 约束的 critic

对每个 prompt，coder 生成一组 K 条程序，sandbox 返回执行结果。Critic 对每条程序预测 pass/fail，并把失败归到七类之一：syntax、import/API、type、assertion、timeout、runtime exception 和 unknown。

系统在组内计算 critic score 与 execution reward 的 Spearman correlation，记作信任系数。相关性为正时，critic-shaped advantage 会增强或削弱相应 rollout 的梯度。Critic 与 executor 不一致时，这项影响会被压低到零。

默认设置使用 K=8 rollouts、Qwen3-8B backbone、rank-32 LoRA 和 400 个 RLVR steps。Coder 与 critic 共享 frozen backbone，并分别更新 adapter。训练数据包含可编译的 SFT examples，以及来自 CodeContests、APPS、SWE-Gym 和 Docker repository edits 的 RLVR prompts。

论文还要求 critic 的文本诊断与 executor failure descriptor 保持一致。因此 critic 同时学习三个目标：pass/fail prediction、正确与错误程序之间的 margin、failure type diagnosis。

## 结果集中在难做 credit assignment 的任务

在 Qwen3-8B 上，vanilla RLVR 在 HumanEval+、MBPP+、LiveCodeBench、BigCodeBench-Hard、APPS 和 CodeContests 六项任务上的平均 pass@1 为 44.9%。ExecuCritic 提高到 48.3%。

SWE-bench Lite 的 resolve rate 从 14.6% 提高到 18.3%。Multi-SWE-bench Python 从 11.7% 提高到 14.4%，Java 从 8.3% 提高到 10.6%。

论文使用 DeepSeek-Coder-V2-Lite 做了第二组 backbone 验证。六项代码任务的平均分从 45.9% 提高到 49.2%，说明收益没有依赖单一模型结构。

提升较大的任务包括 LiveCodeBench、APPS、CodeContests 和 SWE-bench Lite。这些任务生成更长，失败原因也更复杂。Prompted reviewer、Self-Refine 和只做 reranking 的 trained critic 都低于联合训练方案。

三次 RLVR seeds 的 paired confidence intervals 全部高于零。以 SWE-bench Lite 为例，提升为 3.7 points，95% confidence interval 是 +1.9 到 +5.5。

## Critic 也负责节省 sandbox 预算

训练完成后，coder 一次生成 N 个 candidates，critic 先排序，只把 top-k 送入 sandbox。如果全部失败，critic 对最高分 failure 给出诊断，coder 读取这条反馈后再生成一轮。

单次 critic forward pass 通常远便宜于 repository setup 与 test execution。这个区别反映在固定预算实验里：ExecuCritic 只执行两个 candidates 时，六项代码任务平均 pass rate 已达到 45.8%；vanilla RLVR 执行八次时为 44.9%。ExecuCritic 执行八次达到 48.3%，也超过穷举执行 24 个 candidates 的 45.4%。

在 repository tasks 上，每个已解决问题需要的 sandbox calls 从 18.6 降到 10.7。Patch ranking 的 NDCG@5 从 55.8 提高到 66.9。

Critic calibration 也随训练明显改善。它的 score 与 executor outcome 的 Spearman correlation 从 SFT 后的 0.18 上升到 0.71。Validation rollouts 上，ExecuCritic critic 的 ECE 为 0.061，pass/fail AUC 为 72.8，failure type F1 为 64.6，均好于 frozen prompt critic、reranker 和 scalar reward model。

## Ablation 给出的工程信号

完整方案的六项平均 pass@1 为 48.3%，SWE-bench Lite 为 18.3%。移除 critic shaping 后分别降到 45.2% 和 15.0%。取消 calibration gate 后降到 46.0% 和 15.8%。取消 failure diagnosis consistency 后降到 46.7% 和 16.4%。

将 K 从 8 增加到 16，只把平均 pass@1 从 48.3% 推到 48.5%，SWE-bench Lite 从 18.3% 推到 18.5%，sandbox training budget 却翻倍。对 rollout infrastructure 来说，这个结果比单纯扩 sampling 更有吸引力。

数据管线需要保留一组新的 lineage：prompt、coder rollout、executor verdict、normalized failure descriptor、critic probability、group-level calibration coefficient、shaped advantage，以及 refinement round。只有这样，团队才能复查 critic 在哪些 batch 中获得信任，以及错误诊断有没有持续影响 policy。

## 边界

ExecuCritic 仍然依赖可执行 verifier。Documentation、refactoring 和开放式 repository work 缺少稳定 test oracle，需要 synthetic verifier、generated tests 或更弱的 learned judge。

Calibration gate 也只能限制已观测 rollout group 中的错误相关性。Backbone 较弱、任务 out-of-distribution，或者 coder 在训练过程中快速漂移时，critic 仍可能失准。论文没有观察到 reward hacking，但 adversarial critic 与长期 joint-training stability 还需要更多实验。

它给出了一条清晰的 systems 路线：用便宜、可批处理的 critic forward pass，提高 sparse execution reward 的信息密度，再把昂贵的 sandbox 预算分配给最值得验证的 candidates。

## 另外两篇

第二篇是 **ModularRSI: Modular and Generalizable Recursive Harness Self-Improvement**。

它把 coding-agent harness 拆成 Agent Loop、Tool Use、Observation Management、Context Management 和 Task Completion Detection 五个模块。系统对比同一任务的成功与失败 trajectories，在 benchmark-disjoint 的 2,000 个 executable tasks 上寻找跨任务 recurring deficiencies，分别演化各模块，再执行 integration 与 conflict resolution。这个设计适合做 harness change attribution，也方便回滚与跨模型迁移。

第三篇是 **AgentGuard: Learning Execution Guardrails from Anomalous Coding-Agent Trajectories**。

它从真实 coding-agent failure traces 中提炼 instruction-level constraints，并按当前任务动态激活相关规则。在 642 条异常 traces、382 个 repository tasks 构成的数据上，AgentGuard 将 abnormal execution rate 从 69.0% 降到 26.7%，同时把 successful task completion 从 21.7% 提高到 35.0%。

论文：<https://arxiv.org/abs/2609.16604>

另外两篇：<https://arxiv.org/abs/2609.14857>、<https://arxiv.org/abs/2609.16287>
