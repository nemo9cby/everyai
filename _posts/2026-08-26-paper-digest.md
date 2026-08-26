---
title: "Paper Digest: 2026-08-26"
categories: [Paper Digest]
tags: [AI, Agents, Code Agents, Agent Harness, Post-Training, Reinforcement Learning]
---

今天最值得看的 paper，我会选 **AutoSaddler: Automatic Harness Optimization with Durable Updates from Agent Execution Traces**。

长任务里的 agent failure 往往很琐碎。一次工具参数选错，一条 prompt rule 过于宽泛，一个 hook 在错误时机触发，几十步之后才表现成整个任务失败。模型可能足够强，包在模型外面的 harness 仍然会持续制造小事故。

Harness 也很难调。它同时包含 system prompt、tool schema、tool implementation、middleware、agent loop 和 resource budget。开发者通常看几条失败轨迹，补一条规则，再跑一次 benchmark。改动修好当前 case，也可能破坏原本正常的任务。

AutoSaddler 给出一条更系统的路径：把 harness 当作一份可以通过 execution trace 学习和更新的软件 artifact。

## Harness optimization 是一套 offline learning loop

AutoSaddler 先把任务划分成 train、dev 和 test，再用 mini-batch 迭代优化 harness。

每轮包含六个关键动作：

1. 用当前 harness 跑一批任务，收集成功与失败轨迹
2. 同时检查 execution trace 和 harness source，定位 failure root cause
3. 在 prompt、tool、middleware 三层生成结构化 patch
4. 回放同一批任务，确认 patch 是否修复目标问题
5. 在 dev set 上检查泛化与 regression
6. 把 patch、结果和 lessons 写入 EvoDAG，供后续候选组合使用

这套循环和 model training 很像。Failed trace 提供 textual error signal，patch 类似一次 parameter update，mini-batch replay 与 dev evaluation 负责验证更新。区别在于 textual update 缺少自动微分，每次更新都需要明确假设、代码改动和执行证据。

这个定义很重要。Agent 团队经常同时改 model、prompt 和 tools，最后很难解释一次提升来自哪里。AutoSaddler 让 harness update 具备版本、diff、训练样本、验证结果和回归记录，改进过程因此可以审计，也更容易复现。

## 深度 diagnosis 决定 patch 质量

常见的自动优化方法会把失败轨迹交给一次 LLM call，让模型总结原因并改 prompt。AutoSaddler 使用 agent session 主动浏览长轨迹和 harness code，检查具体工具调用、文件与控制逻辑，并比较多个可能的 root cause。

这个过程平均多使用 6.2 次 tool call 和 5.8 次 file access，却能生成更多被接受的 patch。移除深度 diagnosis 后，GAIA2 Pass@1 从 62.0% 降到 57.8%。

长轨迹里的最终错误经常离根因很远。表面现象可能是 agent 没完成任务，真实原因可能是一条 tool description 让模型反复调用错误接口，或者 middleware 在每次调用前注入了不合时宜的提醒。Diagnosis 必须能沿执行链回溯，并把行为证据对应到 harness 实现。

这件事和 debugging production system 很接近。只读最终 error message 很少够用，还需要 event timeline、state transition、tool input/output 和 code path。

## Prompt patch 很容易成为局部最优

AutoSaddler 把 patch 分成三层：

- Prompt，包括 rule addition 与 rule modification
- Tool，包括新增工具、参数修改、implementation fix 与 description fix
- Middleware，包括 pre-tool hook、infrastructure change 与 agent-loop change

它又把这些更新分成 Steering Patch 和 Capability Patch。前者调整文字与行为引导，后者修改 executable code 或 orchestration logic。

论文里有一个很有意思的结果。取消结构化 patch space 后，91.5% 的更新集中在 Steering Patch。模型会不断补 prompt，因为文字修改最容易生成，也最容易解释。真正高价值的 capability change 很少被探索。

结构化搜索改变了这个分布。New Tool、Agent Loop Change 和 Infrastructure Change 的 patch acceptance rate 分别达到 83%、71% 和 67%。Capability Patch 的 fix rate 与 Steering Patch 接近，regression rate 更低，分别为 8% 和 17%。

这对 coding agent 很实用。很多失败无法靠更长的 instruction 解决。模型缺少合适的工具、拿不到必要状态，或者 loop 没有在关键节点做 verification 时，继续堆 prompt 只会增加 context burden。Harness 需要获得新能力，行为才会稳定。

## Regression control 是最关键的一层

一条 patch 在当前 mini-batch 上有效，仍可能过度拟合几条 trajectory。

AutoSaddler 会把回放结果分成 fixed、regressed、still-failing 和 still-passing 四类，并让 Reflection Agent 分析 patch 的作用范围。通过 dev set 的候选才进入长期 evolution memory。EvoDAG 保存每个 harness 的 diff、lesson 和 performance signal，Evolution Agent 可以组合不同 lineage 中已经验证的组件。

移除 generalization-aware selection 后，GAIA2 test Pass@1 从 62.0% 降到 50.6%，这是几个主要 ablation 里最大的下降。研究者展示了一个具体事故：新工具与 hook 修复了目标 case，却强制高频 messaging tool 改道，导致许多无关任务退化。Dev evaluation 和 reflection 会阻止这种 over-scoped patch 留在系统里。

Agent harness 的更新很像修改共享基础设施。局部修复会穿过大量任务路径，回归风险通常高于开发者直觉。可靠的优化系统需要同时统计 fix rate 和 regression rate，并用任务族、repository 或 environment 做隔离验证。

## 三个 benchmark 上的结果

AutoSaddler 在三类 long-horizon environment 上测试：

- GAIA2：53.0% 提升到 62.0%
- SWE-Bench Pro：SWE-agent 从 37.3% 提升到 46.9%
- Terminal-Bench 2.0：Terminus 2 从 40.0% 提升到 50.0%

在 SWE-Bench Pro 中，train、dev、test 使用不同 repositories，减少了记住单一项目约定带来的虚假提升。GAIA2 上用 Opus 4.6 优化出的 harness 换成 Haiku 4.5 后仍有 5.6 个百分点增益，也提供了一点跨模型迁移证据。

它的 compute efficiency 同样值得看。AutoSaddler 大约用 1,000 次 task execution 达到 72.3% dev accuracy，GEPA 与 Meta-Harness 使用约 2,800 次仍停在更低水平。深度 diagnosis 单轮更贵，却减少了大量低价值 search。

## 对 agent post-training 的启发

Model policy 与 harness policy 可以看成两个互相影响的优化对象。Weight update 改变模型在固定环境里的行为，harness update 改变模型能看到的状态、能调用的动作与执行反馈。生产系统很可能需要两条独立 loop，并且记录完整 provenance，避免把 harness gain 误判成 model gain。

对 distributed rollout infrastructure，AutoSaddler 至少提出四个工程要求：

1. Trace schema 要保留 tool call、state transition、artifact、错误与最终 verifier result
2. Policy checkpoint、prompt、tool implementation、middleware 和 environment 必须共同 versioned
3. Candidate patch 要在隔离 worker 上 replay，并对不同 task family 统计 regression
4. Evolution memory 要保存 causal evidence、rollout cost、适用范围与失败历史

这里还有一个很适合继续研究的问题：能否把 harness patch 产生的新成功 trajectory 送回 model post-training，再用更新后的 model 重新优化 harness。两套 loop 如果没有清楚的 evaluation boundary，很容易互相追逐噪声。固定 policy 的 harness search、固定 harness 的 policy training、最后的联合 held-out evaluation，会更容易判断真实收益。

AutoSaddler 最有价值的结论很直接。Harness 是 agent capability 的一部分，也应该拥有训练集、验证集、版本控制、回归测试和优化历史。Prompt 只是其中一层，很多持久改进来自 tool 和 loop 本身。

## 另外两篇

第二篇是 **Paritok-4B: Intent-Conditioned Context Compression for Coding Agents**。

它用 67,074 条真实 OpenHands trajectory 训练一个 extractive compressor，根据当前任务挑选代码和工具输出中的原始 spans，避免改写 identifier、path 和 number。在 SWE-bench Lite 上，Paritok 把 context 压到约四分之一，同时保留 86.5% 的 uncompressed solve quality。264 MB adapter 可以在单张 24 GB GPU 上自托管，很适合放进长时间 coding-agent pipeline。

第三篇是 **Robust Code RL via Faulty-Code-Driven Test case Synthesis and Dense Reward Shaping**。

它处理 code RLVR 里 verifier quality 的核心问题。框架利用 near-correct faulty code 合成更有辨别力的 tests，再通过 validator agents 与 behavioral clustering 去掉错误和冗余测试，并用 pass rate 提供 dense reward。Qwen3-32B 在 LiveCodeBench 上获得 3 个百分点的绝对提升。对 code post-training 团队，最值得借鉴的是用 policy 当前的典型错误来决定下一批 verifier 应该覆盖什么。

论文：<https://arxiv.org/abs/2608.23041>

另外两篇：<https://arxiv.org/abs/2608.24188>、<https://arxiv.org/abs/2608.24135>
