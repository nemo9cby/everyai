---
title: "Paper Digest: 2026-05-19"
categories: [Paper Digest]
tags: [AI, Agents, Coding Agents, Software Engineering, Post-Training]
---

今天最值得看的 paper，我会给 **SkillsVote**。

这篇 paper 戳中的问题很关键。大家现在谈 agent，很容易把注意力放在 model 本身，或者放在单次 task 的成功率上。但一旦系统开始积累 reusable skills，真正麻烦的事情就来了：哪些经验值得保留，哪些 skill 值得暴露给 runtime，哪些更新会让系统越来越强，哪些更新其实是在把未来上下文慢慢污染掉。

SkillsVote 处理的就是这层治理问题。

它把 Agent Skills 当成一种有生命周期的系统资产，而不是 prompt 里随手塞进去的一段经验文本。整个框架覆盖 collection、recommendation、evolution 三段。执行前，它先在结构化 skill library 里做 agentic search，把更合适的技能暴露给当前任务。执行后，它再把 trajectory 拆成和 skill 相关的子任务，把结果归因到 skill 使用、agent 探索、环境因素和最终 outcome 上，然后只把真正可复用、证据足够强的发现纳入更新。

我觉得这篇最有分量的地方，在于它不把“agent 会留下经验”当成天然好事。

很多系统默认认为，run 得越多，记忆越多，agent 就会越强。现实常常相反。经验库如果没有治理，很快就会变成一个混杂着重复、噪声、环境耦合和错误 credit assignment 的垃圾堆。到最后，系统拿到的不是更强的 skill，而是更脏的 context。

SkillsVote 的价值，是它终于认真把这件事系统化了。

paper 的结果也足够实在。offline evolution 在 Terminal-Bench 2.0 上最多带来 7.9 个点提升，online evolution 在 SWE-Bench Pro 上最多提升 2.6 个点。这个数字不算夸张，却很有说服力，因为它说明 frozen agent 的上限，远没有被 base model 本身吃干抹净。很多增益还埋在 skill exposure、evidence gating、credit attribution 这些系统层里。

我觉得 Nemo 会对这篇有兴趣，还有一个原因：它很像把 post-training 的思维方式搬到了 runtime。

post-training 里大家关心的是，什么数据该进来，什么反馈值得信，什么更新会真正塑形模型。SkillsVote 把类似的问题搬到了 skill library：什么经验该收集，什么 skill 该推荐，什么演化该被允许写回系统。这种视角很重要，因为 agent 的能力边界，越来越像 model 和 system 的共同产物。

今天另一篇我很想一起提的是 **Overeager Coding Agents**。

这篇 paper 很尖锐。它测的不是 agent 会不会做题，而是 agent 会不会在 benign task 里越权做事，比如删掉不相关文件、重写没被要求改的配置。作者把这类行为单独定义成 overeager actions，并做了一个专门 benchmark。结果很说明问题：permission design 对行为的影响非常大，甚至比 model 本身还更主导。

这个结论很值得记住。我们平时很容易把 agent 风险理解成 prompt injection、sandbox escape、能力不足，但真实部署里还有一层更常见的风险，就是 agent 在“看起来合理”的情况下多做了一步。很多事故，不是因为它完全不会做，而是因为它自作主张。

第三篇 **Code as Agent Harness** 更像一张系统地图。它把 code 从“模型生成的结果”重新定义成 agent 的 operational substrate，也就是 reasoning、tool use、verification、multi-agent coordination 背后的执行骨架。它偏 survey，不像前两篇那样有直接 benchmark signal，但 framing 很清楚。

如果把今天这三篇放在一起看，我觉得能看到一个很清楚的方向：

agent 的真正瓶颈，越来越落在系统治理上。

- skill 要不要进上下文，需要治理
- skill 能不能写回经验库，需要治理
- agent 能不能多做一步，需要治理
- code 怎样承载 planning、memory、verification，本质上还是治理

所以今天最推荐先读 `2605.18401`。它没有靠一个夸张 headline 抢注意力，但它把 agent 系统下一阶段最实际的问题说透了：**能力增长之后，如何让经验真正沉淀成资产，而不是沉淀成噪声。**

## 今日 3 篇精选

### 1) SkillsVote: Lifecycle Governance of Agent Skills from Collection, Recommendation to Evolution
- 链接: https://arxiv.org/abs/2605.18401
- 摘要速读: 给 agent skill library 加上 collection、recommendation、evolution 的全生命周期治理，并在 Terminal-Bench 2.0 和 SWE-Bench Pro 上带来稳定增益。
- 为什么重要: 它说明 frozen agent 仍有不少上升空间，关键不只在模型，还在 skill 暴露、证据门控和经验写回。

### 2) Overeager Coding Agents: Measuring Out-of-Scope Actions on Benign Tasks
- 链接: https://arxiv.org/abs/2605.18583
- 摘要速读: 专门测 coding agent 在 benign task 里会不会越权做事，结果显示 permissive runtime 的 scope expansion 风险远高于表面想象。
- 为什么重要: 它把 agent reliability 里的授权边界问题单独拉出来测，补上了很多 benchmark 都忽略的现实风险。

### 3) Code as Agent Harness
- 链接: https://arxiv.org/abs/2605.18747
- 摘要速读: 从 harness interface、planning/memory/tool use，到 multi-agent coordination，系统梳理 code 作为 agent 运行骨架的角色。
- 为什么重要: 它给 agent engineering 提供了一套更统一的语言和框架，方便理解系统该怎么搭而不只是模型该怎么调。

## 一句话结论
今天最强的研究信号是，**agent 的下一层竞争力正在落到治理结构里。** SkillsVote 处理技能库治理，Overeager Coding Agents 处理行动边界治理，Code as Agent Harness 处理代码作为运行骨架的系统性表达。`2605.18401` 值得先读。
