---
title: "Paper Digest: 2026-11-24"
categories: [Paper Digest]
tags: [AI, Coding Agents, Reinforcement Learning, SWE-Gym, Data Quality]
---

今天最值得看的 paper 是 **What Makes Good Data for Training Coding Agents? A Systematic Study on SWE-Gym**。

Coding agent 做 RL 时，test suite 通常直接承担 reward function：patch 通过 hidden tests 得 1，失败得 0。大家很容易把训练数据质量理解成 task diversity、difficulty 或 pass rate，却忽略了更基础的问题：这些 tests 给出的 reward 到底对不对？

这篇论文对 SWE-Gym 做了一次 execution-backed audit。它发现大量“模型永远做不出来”的 hard tasks，其实包含会拒绝正确修复的 tests。清理这些错误 reward labels 后，即使严格控制训练集大小和难度，训练出的 policy 在 held-out coding benchmarks 上仍然更强。

## Test suite 就是 coding RL 的 reward model

SWE-Gym 的每个 task 来自真实 issue 和 reference patch。常见的数据构建流程会确认 reference patch 能通过 tests，然后把这个 test suite 用作所有候选 patch 的 verifier。

这个检查只证明 reference patch 被接受，无法证明其他正确实现也会被接受。一个修复可能采用不同 API、不同控制流或更完整的边界处理，功能正确，却被过度绑定 reference implementation 的 test 拒绝。对 RL 来说，这就是 false negative：正确 patch 获得负 reward。

反方向也存在。Tests 覆盖不足时，错误 patch 可能侥幸通过，形成 false positive。

作者没有依赖 LLM judge 直接宣布 verifier 有错。两个 LLM families 先生成多样候选 patch，研究者再检查可疑样本，只有能在原始执行环境中复现的错误才计入 audit。每个 verdict 都保留执行证据。

## “从未解出”经常意味着 reward label 错了

最醒目的结果来自 9B base model：

- 在模型从未解出的 tasks 中，74% 至少有一个 test 会拒绝正确修复。
- 在模型偶尔能解出的 tasks 中，这一比例是 20%。

这说明低 pass rate 混合了两种完全不同的原因：task 确实困难，或者 verifier 在惩罚正确答案。仅按 solve rate 挑选 hard examples，会把大量错误监督送进 RL。

作者修复或移除有问题的 tasks，发布包含 1,087 个 audited tasks 的 SWE-Gym-Verified。为了验证收益来自 label correctness，他们构造相同规模、匹配难度的训练集进行对照。使用 audited tasks 训练的模型，在 held-out SWE-bench Verified 和 SWE-bench Pro 上分别高 2.4 和 2.7 分。

简单删除 base model 从未解出的 tasks 没有带来同样提升。有效操作是检查 reward label，而非按模型表现过滤难题。

## False positives 的结论更谨慎

Audit 还发现，所有通过 tests 的 patches 中，大约四分之一其实是错误修复。作者补充的新 tests 能抓住其中约四分之一，但在当前实验规模下，held-out 分数没有可检测的变化。

这个结果有几种可能解释：新增 tests 捕获的问题对 held-out benchmark 影响有限；训练规模不足以显示差异；或者 false negatives 对策略学习的伤害更直接。它没有证明 false positives 无关紧要。生产系统依然需要防止 agent 利用 verifier 缺口，尤其是会反复采样和优化同一类 reward 的 RL pipeline。

## 对 coding-agent post-training 的实际启发

一条 coding RL data pipeline 至少应该同时追踪四类指标：

- **Reward availability**：task 是否能稳定运行并产生 reward。
- **Reward precision**：通过 verifier 的 patch 有多少真正正确。
- **Reward recall**：正确 patch 有多少会被 verifier 接受。
- **Evidence provenance**：每次修复、删除或保留 test 的判断能否回溯到执行记录。

Pass rate 只能回答第一类问题的一部分。它无法告诉我们 reward 是否正确。

对大规模 distributed RL 来说，这也是 systems efficiency 问题。一个 false-negative task 会消耗 rollout generation、sandbox execution、reward calculation 和 optimizer steps，随后用错误梯度压低正确行为的概率。继续增加 GPU 或并发 worker，只会更快地放大这份错误监督。

更稳健的流程可以先用多个 model families 和高采样温度生成 candidate patches，再对“语义上合理但 test 失败”的样本执行差分检查与人工复核。Audited verdict、环境版本和复现命令应成为 dataset artifact，和 task 本身一起版本化。

## 今天另外两篇相关论文

**Let the Fixer Decide: Training a Code Localizer from the Fixer's Likelihood** 重新定义了 code localization 的训练目标。Reference patch 改了哪些文件，并不等于 fixer 需要读哪些代码。它让 localizer 生成多个 context，由冻结的 fixer 按 verified patch likelihood 打分，再用 weighted SFT 训练 localizer。最终在保持 bug-finding rate 的同时减少 27.5% 输入代码，并减少访问不存在文件的 dead ends。

**Composable Soft-Prompt Distillation of RL Experts** 把不同 RL experts 分别蒸馏成 soft prompts，冻结 base model，并通过拼接 prompts 组合能力。单个 prompt 在 23 个 math、code 和 agent settings 中有 22 个恢复至少 90% teacher gain；完整组合与 multi-teacher on-policy distillation 基本持平，同时保持接近原 backbone 的 throughput。

论文：<https://arxiv.org/abs/2611.30207>

另外两篇：<https://arxiv.org/abs/2611.12164>、<https://arxiv.org/abs/2611.27376>
