---
title: "Paper Digest: 2026-08-25"
categories: [Paper Digest]
tags: [AI, Agents, Code Agents, Multi-Agent Systems, Post-Training, Reinforcement Learning]
---

今天最值得看的 paper，我会选 **Apodex 1.1: Scaling Agentic Intelligence for Complex Work**。

很多 agent benchmark 测的是一次任务能否完成：模型拿到 instruction，调用若干 tools，最后给出答案。真实工作多了几个麻烦。任务会持续很久，文件和外部信息不断变化，中间步骤会失败，多条支线需要并行推进，最终结果还得留下可以检查的证据。

Apodex 1.1 给这种能力起了一个很朴素的名字：working capability，也就是朝着真实目标持续取得可验证的进展。

论文围绕两条 scaling axis 构建这项能力：

- Environment Scaling，扩展 executable file、search 和 code environments 的多样性与可验证性
- Agentic Coordination Scaling，训练 agent 拆解长任务、委派并行工作、整合异步结果，以及在计划失效后重新规划

它最值得注意的地方，是把 model、environment、harness 和 coordination trajectory 放进同一个训练系统里。长时间工作能力由整条执行链共同决定，单独提升某个 benchmark 上的 reasoning score 很难覆盖这些系统问题。

## 长任务需要持续状态

短任务可以把上下文塞进一次 prompt。任务持续到几十分钟甚至几小时后，active context 很快会变成稀缺资源。

Apodex 使用共享 execution harness 和 AgentOS 维护跨工具、跨 agent 的任务状态与 provenance。系统需要知道哪些子任务已经完成，哪个 artifact 来自哪次执行，哪些结论经过验证，哪些分支失败后仍待恢复。

Provenance 在这里尤其关键。多 agent 系统很容易产出一个看似完整的汇总，但主 agent 无法判断每条结论的证据质量。把来源、执行记录和验证结果绑定到 artifact，才能让并行工作真正汇合成可信交付物。

这也改变了 memory 的角色。Memory 不只保存对话摘要，还要保存 task graph、artifact state、tool result、verification status 和 decision history。它更接近一个可恢复的 execution state。

## Environment Scaling 决定训练信号的上限

Agent post-training 需要大量 rollout，rollout 的价值取决于 environment 能否提供真实约束和可靠 verifier。

Apodex 扩展了文件、搜索和代码环境，让模型面对更丰富的工作形态。每个 environment 还需要提供足够强的验证信号。缺少 verifier 时，agent 可能学会生成漂亮的状态报告，却没有稳定完成底层工作。

可执行环境带来更密集的反馈：

- 文件是否按要求生成，结构是否正确
- 搜索结论能否追溯到证据
- 代码能否运行，测试是否通过
- 多个 artifact 能否组成完整交付物
- 失败后的 recovery 是否修复了真实问题

这些反馈让“取得进展”变成可训练的目标。训练系统可以区分忙碌和有效工作，也可以把 failure recovery 留在 trajectory 里，教模型识别错误、修正状态并继续推进。

## Coordination Scaling 需要专门训练

给 agent 增加 subagents 并不会自动产生高质量协作。主 agent 仍然要决定任务怎么拆、哪些分支值得并行、信息如何共享、结果何时汇总，以及冲突怎样解决。

Apodex 把这些 coordination traces 用作训练数据。模型学习的对象覆盖完整协作过程：

1. 把长目标拆成边界清楚的子任务
2. 把任务交给具备合适工具和上下文的 agent
3. 在异步执行期间维护全局状态
4. 检查返回结果的证据与完整性
5. 合并互相依赖的 artifacts
6. 在失败或新信息出现后重排计划

这里存在一个很现实的 credit assignment 问题。最终任务成功可能来自某个关键 subagent，也可能来自主 agent 的及时重规划。只给整条 trajectory 一个终局 reward，会让学习信号变得稀疏。更成熟的 pipeline 需要记录 delegation quality、subtask completion、integration correctness、recovery effectiveness 和最终 verifier result。

## 对 distributed post-training 的启发

这类训练给基础设施提出了比普通 rollout 更复杂的要求。

首先，trajectory 具有异步结构。一个 root task 会展开多个 subagent branches，各分支使用不同工具、耗时和计算资源。Scheduler 需要处理 straggler、失败重试和动态扩缩容，也要避免主 agent 长时间空等。

其次，版本必须贯穿整条执行链。Policy、prompt、tool schema、environment、verifier 和 subagent configuration 都可能变化。缺少版本信息时，训练团队很难解释某次成功来自模型更新，还是 harness 行为发生了改变。

第三，rollout allocation 应该关注 verified progress 和 information value。长 trajectory 消耗大量 token 与 tool compute。系统需要判断哪些失败模式值得继续探索，哪些 branch 已经没有学习价值，以及哪些成功轨迹提供了可复用的 coordination pattern。

对于 heterogeneous cluster，这些问题会进一步放大。搜索、代码执行、GPU inference 和 verifier 可能运行在不同 worker pools。高吞吐依赖的不只是 batching，还包括 task graph scheduling、资源感知路由、状态持久化和 fault tolerance。

## 35B Mini 为什么值得关注

论文报告 35B 参数的 Apodex 1.1 Mini 仍然保留了很强的 working capability，可以本地部署。

这说明 agent 系统的最终表现有很大一部分来自训练分布与 execution system。较小模型如果接受了高质量 environment trajectories、coordination traces 和 recovery examples，再配合可靠 harness，也能在真实工作中形成很有竞争力的组合。

企业部署尤其在意这一点。本地模型可以控制数据边界、推理成本和工具权限。只要工作流有清晰 verifier，35B 级别模型就可能承担长期后台任务，把昂贵 frontier model 留给高不确定性规划或关键检查。

## 这篇论文真正提出的问题

Apodex 1.1 把“agent intelligence”落到一个可操作定义上：持续、可验证地完成复杂工作。

这个定义迫使系统同时面对几个硬问题：environment 是否真实，verification 是否可靠，状态能否恢复，协作是否有效，最终交付物能否追溯。它也给 post-training 提供了更具体的训练对象。模型需要学习的不止是下一步 tool call，还有怎样管理一段有状态、有分支、有失败的工作过程。

对正在做 code agents、multi-agent systems 或 agent RL 的团队，这篇论文值得细读。最有价值的部分未必是某个单项 benchmark 分数，而是它把 environment scaling 和 coordination scaling 放进同一张系统图里。真实的 agent 能力会在这两条轴的交汇处出现。

## 另外两篇

第二篇是 **Prime Agent: A Self-Improving RLM Harness**。

Prime Agent 提供一个开源 long-horizon harness。Persistent IPython REPL 让模型通过程序处理超出 active context 的信息，Continual Harness 保存 histories、memories、skills、prompts 和 subagent specifications，recursive subagents 还能直接通信。它把 recovery、verification 与 resource accounting 做成标准能力，并把 ARC-AGI-3 RHAE Best@1 从 30% 提高到 95.5%。这个结果再次说明，harness 会显著影响我们测到的 model capability。

第三篇是 **ARC: Fair Relative Advantage Comparison in Open-Ended Real-World Interaction**。

ARC 关注 group-based RL 的一个隐蔽假设。同一组 rollouts 可能选择回答、澄清、汇报进度或请求确认等不同策略，它们的 reward 未必适合直接比较。论文通过 strategy-conditioned grouping、hybrid rewards 和 entropy regularization 恢复更公平的 relative advantage，并把 user-visible communication 与 latent tool execution 分开。对训练可交互、可打断的 agent，这个问题很实用。

论文：<https://arxiv.org/abs/2608.23283>

另外两篇：<https://arxiv.org/abs/2608.23552>、<https://arxiv.org/abs/2608.13622>
