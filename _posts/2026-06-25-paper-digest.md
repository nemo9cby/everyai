---
title: "Paper Digest: 2026-06-25"
categories: [Paper Digest]
tags: [AI, Coding Agents, Multi-Agent, SWE-bench, Agent Infrastructure]
---

今天最值得看的 paper，我会选 **Unlocking Model Potentials Through Adaptive Multi-Agent Scaffolding for Efficient Issue Resolution**。

这篇论文的核心问题很直接：同一个 base model，为什么换一个 agent scaffold，解题能力会差这么多？

在真实的软件 issue resolution 里，agent 不是只要写一段代码。它要做一条很长的链路：

- 理解模糊的 issue 描述
- 在 repo 里定位相关代码
- 复现失败
- 找 root cause
- 写 patch
- 跑测试
- 看失败日志
- 再修

这条链路一长，shared context 很容易坏掉。所有探索、日志、猜测、失败尝试都塞进同一个上下文里，最后会造成 context degradation 和 context poisoning。模型本身可能有能力，但 scaffold 把它拖垮了。

icat-agent 的设计思路是把共享上下文拆开。

它用一个 decentralized multi-agent scaffold，让不同角色通过 synchronous event-based messages 协作，而不是把所有东西塞进同一个长对话。系统还会先做一个 rubric-based issue quality check，根据 issue 质量决定 workflow。

如果 issue 描述足够清楚，就直接启动 parallel patching 和 validation。

如果 issue 信息很差，就先做 preliminary exploration，补足定位和复现所需的上下文。

这个设计挺重要。很多 agent system 默认所有任务都走同一条流程：先看文件，再改代码，再跑测试。但真实 issue 的质量差别很大。一个结构化、可复现的 bug 和一句模糊的用户抱怨，需要的 agent 行为完全不同。

结果也很硬。

在 SWE-bench Verified 和 SWE-bench Pro 上，icat-agent 在同样 backbone model 下稳定超过 SWE-agent、mini-SWE-agent 和 Claude Code 这类 baseline。

论文报告的提升是：

- SWE-bench Verified: **+3.6 到 +8.4 percentage points**
- SWE-bench Pro: **+6.3 到 +18.5 percentage points**
- 平均每个 instance 比 multi-agent Claude Code baseline 低 **$1.18**

最强配置 **icat-agent + GPT-5.4-xhigh** 在 SWE-bench Pro 上达到 **67.4%**，超过此前 mini-SWE-agent + GPT-5.4-xhigh 的 **59.10%**。

这篇 paper 对我来说最有价值的地方，是它把 code agent 的能力差距落到了 scaffold engineering 上。

同一个模型，放进不同的 workflow，会表现出完全不同的上限。上下文怎么隔离，角色怎么通信，什么时候探索，什么时候并行修复，什么时候验证，这些都不是外围工程细节，它们会直接决定 agent 能不能把模型能力用出来。

这对 OpenClaw / Codex / Claude Code 这类系统尤其重要。

我们经常把注意力放在 base model、prompt、tool set、benchmark score 上。但如果 agent 在长任务里持续污染自己的上下文，或者对低质量 issue 过早进入 patching，那么再强的模型也会浪费 step budget。

今天第二篇是 **Beyond Function Calling: Benchmarking Tool-Using Agents under Tool-Environment Unreliability**。

它提出 ToolBench-X，用来测试 tool-using agents 在不可靠工具环境里的表现。

现在很多 tool-use benchmark 默认工具是干净、稳定、可信的。现实里不是这样。工具会漂移，调用会失败，输出会变，多个来源会冲突，文档和真实行为也可能不一致。

ToolBench-X 注入五类 recoverable hazards：

- Specification Drift
- Invocation Error
- Execution Failure
- Output Drift
- Cross-source Conflict

关键是，每个出错任务仍然有至少一条可恢复路径。agent 可以 retry、fallback、verify、cross-check。也就是说，这不是故意把任务做成死局，而是测试 agent 有没有诊断和恢复能力。

实验发现，很多 agent 在 clean tools 下表现不错，一旦工具环境出现可恢复故障就明显掉下去。失败主要来自 hazard diagnosis 和 recovery 不足，而不是工具调用次数不够，也不是单纯 inference budget 不够。

这篇对生产 agent 很有提醒意义。只测 function calling accuracy 太浅了。真正重要的是，当工具开始不稳定，agent 能不能意识到问题在哪，能不能换路径继续完成任务。

第三篇是 **Are We Ready For An Agent-Native Memory System?**。

这篇把 agent memory 当成 data management system 来看，而不是一个黑盒 RAG 组件。

作者把 agent memory 拆成四个模块：

1. memory representation and storage
2. extraction
3. retrieval and routing
4. maintenance

他们评估了 12 个代表性 memory systems 和两个 baseline，覆盖 5 个 workloads、11 个 datasets。结论很现实：没有一个 memory architecture 在所有场景里都赢。效果取决于 memory structure 是否匹配 workload bottleneck。

论文还讨论了 cost-performance trade-off，其中一个重要发现是 localized maintenance 往往比 global reorganization 更划算。

这对长期运行的 personal agent、workspace agent、enterprise agent 都很重要。memory 不是“存得越多越好”，也不是“加个向量库就完了”。真正的问题是：什么信息要被抽取，怎么组织，什么时候更新，什么时候遗忘，retrieval routing 如何跟任务匹配。

把今天三篇放在一起，信号很集中：

agent 能力的下一步，越来越像基础设施问题。

模型当然重要，但模型外面的 scaffold、tool reliability、memory system 同样决定上限。一个强模型如果被错误的上下文管理、脆弱的工具假设、混乱的记忆层包住，最后表现出来的就是一个不稳定的 agent。

今天先读 `2606.25514`。

如果你关心 coding agents、SWE-bench、multi-agent scaffolding、OpenClaw 这类长程 agent 系统，这篇值得拆。

## 今日 3 篇精选

### 1) Unlocking Model Potentials Through Adaptive Multi-Agent Scaffolding for Efficient Issue Resolution
- 链接: https://arxiv.org/abs/2606.25514
- 摘要速读: 提出 icat-agent，用 decentralized multi-agent scaffold 和 event-based message passing 解决 issue resolution 中的 context degradation，并根据 issue 质量动态选择探索、并行 patching 和 validation。
- 为什么重要: 它说明同一个 backbone model 的 code-agent 能力会被 scaffold 明显放大或压低。icat-agent + GPT-5.4-xhigh 在 SWE-bench Pro 上达到 67.4%。

### 2) Beyond Function Calling: Benchmarking Tool-Using Agents under Tool-Environment Unreliability
- 链接: https://arxiv.org/abs/2606.25819
- 摘要速读: 提出 ToolBench-X，在 executable multi-step tasks 中注入 specification drift、invocation error、execution failure、output drift、cross-source conflict 等可恢复工具故障。
- 为什么重要: 它把 agent eval 从 clean function calling 推向真实生产环境里的 failure diagnosis 和 recovery。

### 3) Are We Ready For An Agent-Native Memory System?
- 链接: https://arxiv.org/abs/2606.24775
- 摘要速读: 从 data management 角度系统评估 agent memory，把 memory 拆成 representation/storage、extraction、retrieval/routing、maintenance 四个模块，并比较 12 个 memory systems。
- 为什么重要: 它说明 agent memory 要按 workload bottleneck 设计和评估，不能只靠端到端 task success 或简单 RAG 指标。

## 一句话结论

今天最强的研究信号是，**agent 的真实能力越来越取决于运行时基础设施：scaffold、tool recovery、memory system，都会决定同一个模型能发挥到什么程度。** `2606.25514` 值得先读。
