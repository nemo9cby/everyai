---
title: "Paper Digest: 2026-09-28"
categories: [Paper Digest]
tags: [AI, LLM, Distributed Systems, Ray, Data Pipelines, Code Agents]
---

今天最值得看的 paper 是 **RayOrch: Programming and Executing Lineage-Controlled Multi-Grain Dataflows for Foundation-Model Data Preparation**。

基础模型的数据准备，经常同时面对两个方向相反的需求。

PDF、网页和视频会动态展开成 page、region、clip 等子任务，而且每个输入展开后的数量差别很大。GPU 希望把不同输入的子任务混在一起凑满 batch，最大化利用率。结果返回后，系统又要知道每个子任务属于谁、原始顺序是什么、父任务是否已经完成、某次失败应该影响哪些兄弟任务。

Ray Data 和 Daft 可以把数据拍平成 records 再 rebatch。parent ID、顺序恢复、完成判定和失败隔离，往往需要应用层自行维护。RayOrch 把这些关系纳入编程模型和 runtime。

## 把动态 fan-out 变成一等抽象

RayOrch 的程序显式声明两类结构操作：

- `F.expand`：一个 parent 展开成数量动态、顺序固定的 children
- `F.reduce`：按 parent 和 ordinal 把 children 重新聚合

编译器检查 expand 与 reduce 是否匹配。运行时记录每个 child 的 immediate parent、不可变 ordinal、membership 和 terminal state。

这层结构状态与物理执行解耦。batch 可以跨 parent 混合，actor 可以重试或替换，调度和 placement 也可以变化，最终结果仍然能回到正确的 parent 和位置。

这对 foundation-model data pipeline 很关键。一个 200 页 PDF 与一个 3 页 PDF 同时进入 OCR 阶段时，短文档完成后可以立刻进入 assemble 和 upload，无需等待长文档，也无需等全局 regrouping barrier。

## Per-Call FIFO Ready Queue

RayOrch 为每个 Call 维护一条 FIFO Ready Queue，从多个 parents 中取出已经满足依赖的 children，再组成物理 batch。

队列负责提升 GPU batch fill，lineage state 负责保持逻辑结构。二者分开以后，运行时能做到三件事：

1. 跨 parent 凑 batch
2. 每个 parent 独立判断完成
3. 按原始 ordinal 恢复结果

在 controlled ablation 中，加入 1:N rebatching 后，wall time 为 634.1 秒；继续启用 FIFO scheduling 后降到 579.3 秒，减少 8.6%。这个收益来自调度本身，不需要改变模型或 UDF。

## 性能提升来自消除 regrouping tail

RayOrch 在 64 张 NVIDIA H20 上处理 MinerU workload，4,295.7 秒完成 174,744 个有效页面，吞吐为 40.68 pages/s。

与相同 workload 相比：

- 比 Ray Data 少 13.1% wall time
- 比 Daft 少 29.0%
- 比 native MinerU 少 51.6%

强扩展测试中，MinerU 从 4 张 GPU 的 15.26 小时降到 64 张 GPU 的 1.01 小时，达到 15.14 倍加速，也就是理想线性扩展的 94.6%。视频 pipeline 从 8 张扩到 64 张 GPU，RayOrch 达到 7.82 倍加速，Ray Data 为 7.60 倍，Daft 为 6.24 倍。

Docling 的 2,000-PDF workload 使用 4 张 H20，RayOrch 用时 9,489 秒，比 Ray Data 少 16.0%，比 Docling Serve 少 22.3%。

关键区别出现在尾部。Ray Data 和 Daft 在 OCR 结束后还要经历 shuffle、全局 regroup、assembly 和 collection。RayOrch 会在单个 parent 的 lineage 完整后立即释放结果，让 assembly 和 upload 与仍在进行的 OCR 重叠。

## 失败也应该有结构边界

RayOrch 区分普通 infrastructure failure 与 typed `GroupFailure`。

`GroupFailure` 的作用域是一个 Call 和一个 immediate parent。某个 parent 已经确定失败后，runtime 会停止它尚未执行的 sibling grains，同时让其他 parents 继续运行。

论文把 failure 注入 99 个最大的 parents。这些文档只占总数的 5.25%，却包含 51.9% 的 pages。RayOrch 阻止了 6,241 个已经 ready、但结果注定会被丢弃的 sibling calls 进入 UDF，占 poisoned-parent siblings 的 26.5%。三组配对实验中，wall time 平均减少 14.93%，未受影响的 parents 仍然产出完整结果。

这个设计对昂贵的 VLM preprocessing 很实际。发现一份 PDF 已经无法组装后，继续 OCR 它剩余的几十页只会消耗 GPU。

## 对 Ray 数据流水线的启发

RayOrch 最有价值的地方，是把 lineage 从日志和调试信息提升成 execution contract。

对 multimodal training-data pipeline，可以直接检查几个问题：

- 动态 fan-out 后，parent、ordinal 和 membership 由谁维护
- 一个 parent 完成后，能否立即进入下游 stage
- batching 是否能跨 parents，同时保持有序恢复
- retry 或 actor replacement 后，旧结果是否可能重复 commit
- 一个 parent 已经失败时，还有多少 sibling work 被继续执行
- end-to-end tail 有多少来自全局 regrouping 和 collection

这篇 paper 和 Nemo 的工作交集很直接：Ray、异构分布式计算、foundation-model data preparation，以及系统语义与 GPU 利用率之间的权衡。值得把现有 pipeline 的全局 regroup barrier 单独拉出来做一次 ablation。

## 今天另外两篇相关论文

**Analyzing and Mitigating Cost-Inefficient Behaviors in Coding Agents** 分析 Claude Code 与 Mini-SWE-Agent 的 1,200 条轨迹，发现 subsumed retrieval、重复生成相似脚本和重复跑测试出现在 79% 到 98% 的任务中，最多占任务成本的 22.75%。Developer-designed skills 在 held-out tasks 上最多降低 41.73% 成本，效果约为 agent-synthesized skills 最佳结果的两倍。

**SLCA-GRPO** 针对 tool-calling RL 的跨 segment credit contamination。它把 execution advantage 路由到 tool tokens，把 preference advantage 路由到 summary tokens，避免最终自然语言回答的噪声更新工具选择与参数。7B 模型在相同训练预算下，相对 GRPO、ToolPO 和 RLTR 分别在 in-domain、BFCL 与 tau-squared Bench 上取得 2.53、1.36 和 9.15 points 的提升。

论文：<https://arxiv.org/abs/2609.18703>

代码：<https://github.com/OpenDCAI/RayOrch>

另外两篇：<https://arxiv.org/abs/2609.30725>、<https://arxiv.org/abs/2609.29050>
