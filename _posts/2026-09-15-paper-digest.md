---
title: "Paper Digest: 2026-09-15"
categories: [Paper Digest]
tags: [AI, LLM, Foundation Models, Post-Training, Code Agents, RL]
---

今天最值得看的 paper，我会选 **ZGCM-1: A Fully Open and Extremely Efficient Foundation Model for Math and Agentic Search**。

这是一份少见的全链路 7B foundation model 技术报告。作者开放了 pre-training、mid-training、SFT 和 post-training 各阶段权重，也开放训练代码、数据配方、intermediate checkpoints、W&B logs 与 evaluation harness。

它最有价值的地方在于把 architecture、training system、data curriculum 和 agent trajectory 放到同一套实验里。模型只有 7.39B 参数，却支持 256K context，并在数学、web research 和 binary analysis 上给出了一组有竞争力的结果。

ZGCM-1 在 AIME 2026 上得到 75.0%，WebWalkerQA 为 63.1%，Binary Function Search 为 62.0%。最后一个任务要求模型使用 Ghidra 工具，在 stripped ELF binary 里找到描述对应的函数入口。它明显超过同尺寸模型，并接近 GLM-5.1 的 66%。

## 256K context 的系统账

模型有 32 层，其中 27 层使用 128-token gated sliding-window attention，5 层使用 global attention，比例为 5:1。

这个结构的收益在长 context 下很直接。相对 32 层 full attention，ZGCM-1 在 256K context 上报告：

- training throughput 提高 3.94 倍
- KV cache 从 32 GiB 降到 5 GiB
- 每个 token 的 KV footprint 从 128 KiB 降到 20 KiB

在一个共同的 10B-token architecture sweep 中，SWA 5:1 达到 9,566 tokens/s/GPU，tail loss 为 1.93，与 full-attention baseline 相当。MLA 的 throughput 为 7,645 tokens/s/GPU，也没有带来对应的 loss 改善。

训练侧使用 hybrid FP8、Muon optimizer 和 TWEO outlier regularization。论文报告的 16K pre-training time-to-loss 相对 BF16/AdamW baseline 提高约 4.2 倍，其中 FP8 贡献约 1.5 倍，Muon 贡献约 1.8 倍。

这种结果很适合用来提醒团队：长 context 能力不能只看模型结构图。Attention schedule、KV footprint、precision、optimizer、parallelism 和 kernel 实现共同决定最终能否训练和服务。

## MDP mid-training

ZGCM-1 的 general pre-training 总计约 4.19T tokens，之后再进行 600B-token mid-training。Context length 分阶段扩展：16K、64K、256K。

更值得注意的是 agent traces 的处理方式。作者将 interaction trajectory 重写为 Markov Decision Process 的 state-action transitions，在 mid-training 阶段提供更密集的 step-level supervision。Agentic data 包括 deep research、software engineering 和 terminal interaction，模型需要学习 action selection、tool call、observation consumption 与 final response。

长 context curriculum 也保留大量短序列。64K 阶段包含 180.89B 个不超过 16K 的 tokens，以及 59.11B 个 16K 到 64K 的 tokens。256K 阶段仍有 127.81B 个不超过 16K 的 tokens，另有 21.72B 个 16K 到 64K 的 tokens，以及 30.98B 个超过 64K 的 tokens。

这个细节很重要。直接把训练数据拉成长，容易让模型失去短任务上的密度与优化效率。ZGCM-1 把 context scaling 设计成 curriculum，同时保留较短样本作为能力底座。

## SFT 数据质量比数量更有用

SFT corpus 包含 4,921,933 个 examples，覆盖 instruction following、knowledge、math、science、code、dialogue、reasoning 和 agentic interaction。

作者先用 deterministic rules 清理结构问题，例如缺失 turns、错误 roles、duplicate generations、prompt-template leakage 和 malformed reasoning traces。随后使用基于 human-annotated pilots 校准的 evaluator，对 educational value、logical soundness、reasoning coherence、factual consistency 和 safety 做分层评分。

论文报告了一个很实用的结果：删掉约 50% raw SFT candidates 的 aggressive tiered filtering，在 aggregate score 上优于未经整理的完整语料。单项 benchmark 并非全部上升，因此这仍然是 mixture 与 capability trade-off，而非简单的“删得越多越好”。

另一个结论与 reasoning data 有关。Long-CoT 比例过高会产生 verbosity bias，并伤害 instruction adherence。最终 recipe 把 think examples、direct-response examples 和 strict-format instructions 混合训练。Intermediate checkpoints 显示，结构化 reasoning supervision 还能提高 no-think response 在 code、math 和 logic 上的准确率。

## Agent data 需要与运行环境精确对齐

Agentic SFT 有三个分支：deep research、software engineering 和 terminal interaction。

Deep research 保留 20,217 条 multi-step trajectories。Software engineering 保留 30,014 条 execution-grounded trajectories 和 30,000 条 execution-free trajectories。前者训练真实 repository interaction，后者覆盖 repository understanding、file localization、tool selection 和 patch planning。

Terminal trajectories 在 isolated Docker environments 中运行，使用 persistent shell 和 structured bash tool，最后由 environment-side verifier 判定结果。

作者发现，training schema 与 inference environment 的不一致会明显降低 agent performance。这里的 schema 包括 system instruction、message roles、tool definitions、structured calls、observation placement 和 final-answer convention。

因此数据管线需要验证：

- tool arguments 是否满足结构约束
- action 与 observation 是否一一配对
- call IDs 是否唯一
- assistant trajectory 是否正确终止
- 自动修复是否拥有无歧义的 call-result 对应关系

无法可靠修复的样本直接丢弃。每条样本保留 stable ID，用于串联 source record、quality decision 和 protocol revision。

作者还比较了两种 schedule。先训练 general data，再继续训练 agentic data，表现弱于把两类样本持续 interleave 的 joint schedule。最终模型采用 joint training。

## 256K SFT 与 mixed GRPO

Released SFT checkpoint 使用 262,144 最大长度，训练 10 epochs，总计 19.46B packed tokens，global batch size 为 48。

Loss 只作用于 assistant reasoning、responses 和 structured tool calls。System/user messages 与 tool observations 保留在 context 中，同时被 mask 掉。这个设计让模型学习 action selection，也避免把环境返回内容当作需要复现的 target。

随后进行 mixed RL，数据覆盖 mathematics、code 和 general tasks。Math 使用 binary correctness reward，code 使用 executable tests passed fraction。GRPO rollout 有两种规模：每步 24 problems × 16 responses，共 384 trajectories；或者 384 problems × 8 responses，最高 3,072 trajectories。单条 response 最长可生成 65,536 tokens，总 prompt-response budget 为 98,304 tokens。

论文对 RL 的结论相对克制。Mixed RL 在 math 与 code 等单领域任务上继续带来收益，但通用 software engineering 和 terminal agency 仍然较弱。作者明确把 SWE-bench Verified、Terminal-Bench 2.0 和 end-to-end interactive agentic RL 留作下一步。

## 对 post-training infra 的启发

这篇报告提供了一张很完整的 data lineage 图。

一条 agent trajectory 从 raw collection 开始，经过 environment verification、schema normalization、tool-protocol validation、quality tiering、benchmark decontamination、length routing、packing 与 loss masking，最后进入 SFT 或 RL。Stable example ID 应该贯穿这条链路。

对分布式训练平台来说，至少要能回答这些问题：

- 这个 packed sequence 来自哪些原始 trajectories
- 哪些 tokens 参与 loss，哪些只是 observations
- execution verifier 在哪个环境版本上运行
- schema conversion 修改了哪些 fields
- 质量筛选为何保留或删除一个样本
- rollout、reward、intervention 与 policy checkpoint 如何对应
- capability gain 是否伴随 verbosity 或 instruction-following regression

ZGCM-1 仍有明显边界。7B 参数限制了 closed-book knowledge；agent behavior 对 tool schema 和 environment feedback 很敏感；通用 repository 与 terminal tasks 的完成率还不高。报告中的强 benchmark 结果也需要在开放 harness 和 checkpoints 上独立复现。

但它已经提供了很好的实验起点。完整开放的训练链路，比单独发布一个 final checkpoint 更能帮助研究者判断：能力究竟来自 architecture、data mixture、execution verification、SFT objective，还是 RL rollout budget。

## 另外两篇

第二篇是 **Dream-RSI: Recursive Self-Improvement through Evolving Worlds**。

它把 coding agent 的历史 discovery trees 变成 replay simulator，在已有搜索空间里低成本评估和改进 exploration policy。底层 coding model 保持不变，orchestration layer 离线“做梦”，再把新 policy 放回 algorithm engineering、math optimization 和 GPU kernel engineering 的真实环境中。这个设计可以减少昂贵 online evaluations，不过也需要警惕 replay coverage 带来的 simulator bias。

第三篇是 **MInTRL: Off-policy Intervention can boost On-policy RL**。

它让 judge 在 on-policy rollout 中偶尔检查当前输出，只替换一个短小的错误 suffix，然后把控制权交还给原 policy。这样可以把 learner 带到有限 rollout budget 很难自己发现的正确 continuation，同时保留大部分 on-policy tokens。实验显示 moderate intervention intensity 最有效，并且 self-intervention 也能带来收益。

论文：<https://arxiv.org/abs/2609.13356>

另外两篇：<https://arxiv.org/abs/2609.14858>、<https://arxiv.org/abs/2609.12419>
