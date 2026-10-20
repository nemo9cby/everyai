---
title: "Paper Digest: 2026-10-20"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Reinforcement Learning, Post-training, SWE]
---

今天最值得看的 paper 是 **Self-Evolving Coding Agents via Harness-Policy Optimization**。

它讨论了 code agent 训练里一个经常被拆开处理的问题：最终能力由 policy model 和 agent harness 共同决定。Harness 包含 system prompt、工具接口、runtime logic 和 interaction workflow。它决定模型能产生哪些 trajectory，policy update 又会改变下一轮真正需要解决的 failure modes。

Self-HPO 把两者放进一个递归优化循环。实验以 Qwen3.5-35B-A3B 和 mini-SWE-agent 为起点，平均 resolve rate 从 57.8% 提高到 64.8%。在 SWE-Bench Verified 上，成绩从 61.0% 提高到 69.5%；用 Opus 4.8 充当外部 harness proposer 时进一步达到 71.6%。

## H-step：让当前 policy 分析自己的失败

每轮先冻结 policy，在 search set 上执行任务并收集 failed trajectories。当前 policy 随后充当 harness proposer，阅读这些失败，修改 prompt 或 executable runtime logic，连续提出并评估六个候选 harness，保留得分最高的版本。

Search set 也会随 policy 演化。初始集合从训练数据随机采样；完成一次 policy update 后，所有 rollouts 都没有解决的任务会组成下一轮 hard-task set。新的 policy 在这些任务上重新生成失败证据，下一次 harness search 因而针对当前 checkpoint 的短板。

这个设计避免了一个常见问题：同一套 harness 可能只擅长补偿旧 policy 的弱点。模型经过 RL 后，失败分布已经改变，继续对最初那批案例做 harness tuning，很容易优化过期目标。

## P-step：在更新后的 harness 下训练 policy

选出新 harness 后，系统冻结 harness，使用它收集 on-policy trajectories，再以 executable verifier 的 binary outcome 训练模型。

默认优化器结合 GRPO 和 Balanced Adaptive Rollout。BAR 会为稀疏 outcome reward 动态补充 rollouts，减少 group 内全零或全一造成的 zero-variance samples；GRPO 再使用 group-relative advantage 更新 policy。

两次 H/P 交替之后，Self-HPO 在三套 benchmark 上都超过 policy-only 和 harness-only baselines。SWE-Bench Verified 达到 69.5%，比 RLVR 高 2.0 points，比 Self-Harness 高 4.3 points。训练任务全部来自 Python projects，但收益还能迁移到包含九种非 Python 语言的 SWE-Bench Multilingual，以及覆盖更广泛 repository tasks 的 SWE-Bench Pro。

## Harness 是 post-training system 的一部分

固定 harness 的 RL 只能学习这套 workflow 能诱导出来的行为。固定 policy 的 harness search 也只能重新组合模型已有能力。Self-HPO 的关键价值，在于让两种优化互相改变对方的数据分布。

论文还交叉组合了 base 与 optimized harness/policy。两个组件单独替换时都保留增益，最终组合的提升超过单项提升之和。结果也同时改善 pass@1 和 pass@k，说明它既提高单次运行的可靠性，也扩展了 agent 能解决的问题边界。

对 code-agent 团队，这里有三个直接可用的设计：

1. 把 harness 与 policy checkpoint 作为一对版本化 artifact。
2. 用每轮 RL 中始终未解决的任务驱动下一轮 harness search。
3. 保存 H-step、P-step 与 verifier outcome 的完整 lineage，避免把 harness gain 错算成 model gain。

这套循环很适合分布式 rollout 系统。Ray actors 可以并行评估候选 harness，复用 repository snapshots，并把 unresolved tasks 动态路由到下一轮 search set。更进一步，还可以把 harness change 拆成 tool protocol、context policy、verification loop 等 primitives，研究哪些改动能跨 model checkpoint 复用。

## 今天另外两篇相关论文

**CoworkEnv: Synthesizing Training Environments for Long-Horizon Cowork Agents** 生成可执行的 coworker training environments，把用户请求、service records、工具、workspace state、workflow constraints 和 programmatic graders 组织在一起。它为 SFT 生成 teacher demonstrations，也为 RL 生成 student rollouts。Qwen3-VL-8B-Instruct 在 Claw-Eval 上从 46.6 提高到 64.1，多轮任务提升 31.9 points。

**Evidence-First Agentic Verification** 提出 Sauron，在接受 agent 的完成声明前，独立检查 environment state 与 execution trace，并为每个 rubric verdict 生成可审计的 evidence receipt。GPT-5.6 Sol 版本达到 0.96 F1，同时比 strongest baseline 低 23% 成本；在 13 类 fault injection 中检出 97% 的失败，只误报 2% 的 clean controls。

论文：<https://arxiv.org/abs/2610.22791>

另外两篇：<https://arxiv.org/abs/2610.24960>、<https://arxiv.org/abs/2610.17724>
