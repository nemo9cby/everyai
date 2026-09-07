---
title: "Paper Digest: 2026-09-07"
categories: [Paper Digest]
tags: [AI, Search Agents, Post-Training, SFT, Reinforcement Learning, Context Management]
---

今天最值得看的 paper，我会选 **Iris: Climbing to the Search Frontier**。

Iris 训练了两个 search agent：35B-A3B 的 Iris-mini，以及 397B-A17B 的 Iris-pro。论文的价值不只在 benchmark 分数。它把 task synthesis、trajectory filtering、SFT、online RL、长尾 rollout 调度和 context management 串成了一条完整训练管线。

其中最值得拆开的机制叫 **SFT-RL climbing**。每轮 RL 结束后，系统挑出当前 policy “已经能解，但成功率还不超过 50%”的问题，再从成功样本里选最短、同时达到最低搜索深度的 trajectory，回流到下一轮 SFT。RL 负责探索稀有成功路径，SFT 用 dense token supervision 放大这些路径。随着模型变强，入选问题的难度也会自然提高。

## 先制造真正需要搜索的问题

很多 web-search benchmark 可以被关键词匹配绕过。Iris 从网页链接结构中抽取 connected entity graph，让生成器沿多跳关系编写问题，然后把问题里所有非答案实体改写成描述性线索，移除名称和 alias。

一道题只有同时满足两个条件才会进入训练集：

1. Reference model 在 closed-book 条件下答错，说明题目足够难。
2. 给出 supporting entity graph 后答对，说明题目可解且答案明确。

这个 hard-but-solvable gate 很实用。它把“更难的数据”具体化成可验证条件，也减少了错误标注和无解问题进入 rollout 集群的概率。

## Trajectory filtering 做了两层

Teacher 在 live search 环境里生成 ReAct trajectory。第一层 coarse filtering 删除答案错误、搜索深度不足和退化样本。重复循环通过 sliding-window compression ratio 检测，因为重复文本经过 zlib 后会异常容易压缩。系统也检查重复行、长字符串和参数完全相同的连续 tool call。

第二层 fine filtering 处理局部坏 turn，例如多余搜索、虚构 tool name，或者 reasoning 与 action 不一致。Judge 根据数据中反复出现的 failure mode 归纳 rubric，对每个 assistant turn 输出 KEEP 或 MASK。被 mask 的内容仍留在 context 中，却不参与 loss，且每条 trajectory 最多 mask 10%。这样可以保留真实 interaction history，同时减少坏动作的监督权重。

## 长尾 rollout 怎么避免拖住同步训练

Search trajectory 的长度有明显 heavy tail，少数 session 会拖慢整个 synchronous step。Iris 使用 request-level partial rollout：当一个 step 已经收集到足够多的完整 trajectory，仍在运行的 oversampled request 会被中断，下一步从 committed prefix 恢复。

Rollout 被保存成一棵 message-state forest。节点缓存 token、loss mask、log probability 和 policy version delta。恢复后的 trajectory 可能拼接不同 policy version 生成的 prefix，系统用 truncated importance sampling 修正 mismatch。

这套设计以大约 2 倍 oversampling headroom 换取更高的 rollout GPU 利用率，也保留了已经完成的 turn。对于同步 RL 集群，这是一个很具体的 straggler 处理方案。

Reward judge 和 observation summarizer 都由训练集群内的 Qwen3.5-397B-A17B FP8 engine 提供。同一组服务既判断最终答案，也把网页压缩成 query-relevant observation，训练不依赖外部 API。

## Context management 必须单独报告

论文在固定 tool、context limit 和 judge 的条件下，同时报告有无 context management 的结果。Iris-mini 在 BrowseComp 上从 64.7 提升到 82.2，context management 贡献 17.5 分；某些更激进设置的增益达到 21.2 分。

这意味着 search-agent 分数里可能混合了 policy capability 和 inference harness capability。Iris 的底层 policy 在关闭 context management 后仍明显领先多组 baseline，开启后优势继续保留，因此训练收益与 harness 收益可以被分别观察。

最终，Iris-mini 在 BrowseComp、BrowseComp-ZH、DeepSearchQA 和 HLE 上得到 82.2、84.8、86.9、52.3；Iris-pro 得到 88.6、85.1、92.9、56.4。所有结果来自 single ReAct agent，没有 sub-agent，也没有 test-time verification。

## 另外两篇

第二篇是 **RISE: Recursive Improvement via Self-Extrapolating Policy Distillation**。

RISE 用当前 checkpoint 与 trailing anchor 之间的位移构造 synthetic teacher，把 RLVR 的 sparse outcome signal 转换成 dense token target。Teacher 会随着 policy 更新持续刷新，不需要更强的外部模型或 privileged conditioning。实验覆盖数学、STEM、code generation 和 multi-turn agent task。

第三篇是 **Group Adaptive Clipping Policy Optimization**。

GAPO 关注 GRPO 的 fixed importance-ratio clipping。低成功率问题里的罕见正确 rollout 往往有更大的 IS ratio 和更强的探索信号，也更容易被固定边界裁掉。它根据 rollout advantage 调整 clipping boundary，在 Qwen 与 Llama 的数学和 coding benchmark 上同时改善 Pass@1 与 Pass@k。

论文：<https://arxiv.org/abs/2609.04304>

另外两篇：<https://arxiv.org/abs/2609.05295>、<https://arxiv.org/abs/2609.00444>
