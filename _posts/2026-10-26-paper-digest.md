---
title: "Paper Digest: 2026-10-26"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Context Compaction, Agent Systems, Post-training]
---

今天最值得看的 paper 是 **Compact-Bench: Measuring What Context Compaction Costs Long-Horizon Coding Agents**。

长任务跑到 context window 边缘时，coding agent 通常会把完整历史压成一份 summary，再带着 summary 和当前 repository state 继续工作。这个过程看起来像普通的信息压缩，实际决定了 agent 能不能完成下一步。

Compact-Bench 给出了一个直接的测量方法：从相同的历史和 repository snapshot 出发，一条分支保留 full history，另一条分支只保留 compacted summary，然后让同一个 continuation agent 继续执行。两条分支的差异可以归因到这次 context replacement。

## 三种互补的评估

Benchmark 包含三条评估轨道。

第一条是 **Task Preservation**，使用 35 个来自 7 个 repositories 的 SWE-bench repairs。作者先验证 full-history continuation 是否能完成修复，再检查 summary-only continuation 是否仍能通过同一组测试。

第二条是 **Recurring-Detail Retention**，覆盖 146 个真实或模拟的 coding-agent sessions，并从后续请求中提取 9,330 个会再次出现的细节，包括 paths、commit SHAs、PR IDs、error types、URLs、key-value pairs 和 IP addresses。

第三条是 **Production-Chain Retention**，分析 106 个真实 sessions 中的 224 次 compaction boundaries。它观察 agent 连续经历多次 history replacement 后，早先的有效状态还能留下多少。

这三条轨道分别回答任务能否继续、具体信息留下多少，以及反复压缩会怎样改变 working state。

## 同一个任务，summarizer 可以决定成败

35 个 repairs 中有 22 个在 full-history 条件下成功。固定 Claude Code 的 compaction prompt 后，14 个 summarizer backbones 对这些成功任务造成的稳定损失率从 5% 到 64% 不等。

Claude Opus 4.8 保留了 21/22 个 repairs。Kimi-K3 和 GPT-5.6-Luna 各保留 18 个，DeepSeek-Flash 只保留 8 个。14 个 backbones 只在 4 个 repairs 上完全一致，其余 18 个都出现分歧。

这说明 compaction model 是 agent runtime 的关键组件。主模型、repository 和后续执行预算都没有改变，仅仅替换 summarizer，就会改变修复是否完成。

Prompt 同样重要。固定 Kimi-K3 后，Kimi Code prompt 保留 19/22 个 repairs，Claude Code 为 18/22，Gemini CLI 为 17/22，Codex 和 opencode 都是 16/22。

## 保留关键词，不等于保留工作进度

论文最值得注意的结果来自 detail retention。

最好的 LLM summary 只保留 16.0% 的 recurring details。一个机械式的 `anchor_index` extractor 不生成自然语言，只从历史中抽取结构化 identifiers，却保留了 79.1%。真实 production summaries 在第一次 compaction boundary 的平均 retention 为 15.7%。

然而更高的 detail retention 也不保证任务成功。Kimi Code prompt 完成 repairs 的成绩最高，recurring-detail retention 只有 8.2%；Claude Code prompt 保留了 15.0% 的 details，repair 成绩略低。

作者进一步检查四个稳定失败案例。每份 summary 都保留了 target file，其中一个 pytest repair 的 5 个关键 patch tokens 全部存在，continuation 依然失败。Sphinx 案例也保留了相关字符串，却丢失了为什么要把 `split(None, 1)` 改成 `rsplit` 的推理进度。

Context compaction 真正需要保存的是可执行的 working state：已经验证了什么假设，哪些路径排除过，当前 blocker 是什么，接下来应当进行哪个 edit 或 test。关键词只是这份状态的局部投影。

## 对 agent systems 的直接启发

1. Compaction 需要 paired continuation evaluation。静态 summary quality 无法代替后续任务是否完成。
2. Structured extraction 可以保留 paths、errors、commits 和 identifiers，适合作为生成式 summary 的稳定补充。
3. Pending hypotheses、completed checks、repository mutations、blockers 和 next actions 应进入显式 state schema。
4. Summarizer、compaction prompt、continuation agent 和 evaluator version 都应记录在 trajectory lineage 中。
5. 连续多次 compaction 要单独测试，因为每一次 history replacement 都可能重新解释已经压缩过的状态。

对于长轨迹 agent training，这还意味着训练样本中的 summary 不是中性输入。它会改变 policy 看到的 state distribution，也会把基础设施造成的状态损失混进 policy failure。收集 rollout 时，应当把 compaction configuration 当成环境版本的一部分。

## 今天另外两篇相关论文

**RSIHub: A Composable Framework for Agent-Evolution Methods** 把 prompts、skills、tools 和 orchestration 建模成显式 mutable resources，并为 agent evolution 提供共享的 execution、evaluation、recovery 和 lineage。它复现 AHE、Hyperagents、A-Evolve 与 GEPA，最大观察到 13.5 个百分点的 pass@1 提升，同时发现 method ranking 会随配置变化，优化收益也不总能迁移到 held-out tasks。

**On-Policy Self-Training: Data-Free Self-Improvement in Large Language Models** 让模型仅从 coarse topic descriptions 出发，自行生成 curriculum、questions、answers 和 privileged reasoning traces，不依赖人工 prompts、labels 或更强 external teacher。论文在 text 和 multimodal domains 上报告了持续提升，也把 synthetic curriculum quality 和 self-confirmation bias 推到 post-training 设计的中心。

论文：<https://arxiv.org/abs/2610.31710>

另外两篇：<https://arxiv.org/abs/2610.32253>、<https://arxiv.org/abs/2610.32770>
