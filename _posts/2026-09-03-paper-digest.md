---
title: "Paper Digest: 2026-09-03"
categories: [Paper Digest]
tags: [AI, Agents, Skills, Coding Agents, Post-Training, Operational Knowledge]
---

今天最值得看的 paper，我会选 **Repo-To-Skill: Distilling GitHub Repositories Into AI4AI Skills**。

Agent 做 ML research 时经常卡在一些很具体的地方：某个 package 的 API 怎么组合，training config 有哪些隐含约束，evaluation script 如何跑通，失败后该检查什么。模型也许理解方法，harness 也具备 planning、memory 和 tool use，但这些 operational knowledge 仍散落在 repository、文档、issue 和脚本中。

Repo-To-Skill 提出 DisCo，把这些知识蒸馏成可以被 agent 直接调用、经过验证、可以长期复用的 skill graph。论文进一步构建了 AREX-Skill Library：从 1,000 个常用 ML repository 中得到 5,353 个 skill，覆盖 20 个领域与 178 个 capability family。

最关键的实验控制得很干净。GPT-5.5 backbone、Codex harness、reasoning effort 和 downstream execution budget 保持一致，唯一变量是 agent 能否访问 distilled skills。

结果很大：

- MLE-bench Any-Medal：31.11% → 72.89%
- PaperBench：29.45% → 39.59%
- FrontierCS：70.63 → 77.14
- PassNet AS Score：1.343 → 1.5313

## Skill 的三个组成部分

每个 skill graph 分为三层。

第一层是 `SKILL.md`。它是 agent 最先读取的入口，说明适用范围、standard operating procedure、tool usage、worked example、failure mode，以及何时继续读取更深的内容。

第二层是 `references/`，保存 API 文档、算法细节、parameter configuration 等知识。Agent 只在任务需要时加载对应部分。

第三层是 `scripts/`，提供输入输出明确的 executable wrapper。高频操作可以直接运行，减少 agent 临时重写和反复调试。

一个大型 repository 往往会被拆成多个 skill。入口 skill 负责路由，component skill 分别覆盖 setup、training、evaluation、diagnosis 或 repair。Progressive disclosure 让 agent 只把当前需要的分支放进 context，几千个 skill 也无需一次性加载。

## 蒸馏包含四个阶段

DisCo 的 skill distillation 依次回答四个问题：

1. Scope：这个 source 或 task 真正需要哪些 capability？
2. Ground：哪些 source evidence 能支持这些 capability？
3. Construct：怎样包装成 procedure、reference 与 executable interface？
4. Verify：skill 是否能通过测试，失败如何修复，仍有哪些 unresolved gap？

论文支持两种工作方式。

Task-agnostic distillation 以 repository 或 paper 为起点，提前构建可长期复用的 skill。Task-oriented distillation 以具体问题为起点，先做 task decomposition 和 capability gap analysis，再主动搜索 source，生成当前任务需要的 skill。

Verification 是整个设计的硬边界。Repository skill 会使用 native example、test、CLI check、tiny fixture 或 smoke script 验证。失败会触发局部修复与重跑，无法消除的缺口则记录在 construction record 中。论文为 1,000 个 repository 分配的平均构建成本约为每个 40 美元，这是一笔离线成本，之后由多次任务摊薄。

## 固定 harness 后，skill 的净增益有多大

MLE-bench 覆盖 75 场 ML competition。加入 task-oriented skill graph 后，Any-Medal 从 31.11% 提升到 72.89%，绝对增加 41.78 个百分点。High difficulty subset 从 13.33% 升到 62.22%，提升尤其明显。

PaperBench 的 20 个 paper replication task 使用 related work 生成 paper-derived skills，同时排除 target paper 自身及其 released code。平均分从 29.45% 提升到 39.59%，20 个任务中有 18 个改善。

FrontierCS 的 188 个 agent task 使用同一个 recovery-oriented skill graph，分数从 70.63 提升到 77.14。低于 50 分的 47 个任务提升最大，平均分从 19.43 升到 45.99。Skill 在 agent 已经找不到可行路径时最有价值。

PassNet 要求 agent 编写 graph compiler pass，同时满足 correctness 与 speed。加入 skill 后，correctness 从 81.35% 提升到 90.76%，失败样本从 14 个降到 5 个。

## 这套方法仍有成本与失败模式

Skill 会让 agent 做更多有效探索。FrontierCS 中，平均 token 使用从每个任务 2.46M 增加到 4.47M，step 和 tool call 也同步上升。论文发现额外 usage 与 task-level gain 几乎没有相关性，因此提升无法用单纯的 test-time compute 增长解释，但部署者仍需支付更多 runtime cost。

Routing precision 也会出错。PaperBench 有两个任务在加入 skill 后退步，论文认为 retrieved skill 可能把 agent 引向一个通用但不适合当前 paper 的实现路径。更好的 router，以及低匹配度时允许 agent 回到 unguided exploration，会是下一步。

另外，公开的 5,353 个 repository skill 与四个 benchmark 的实验 skill 并非同一批资产。MLE-bench、FrontierCS 和 PassNet 使用 task-oriented graphs，PaperBench 使用为 target 相关工作单独构建的 paper-derived pool。看到 library scale 与 benchmark gain 时，需要保留这一区分。

## 对 agent infrastructure 的启发

这篇论文把 agent 系统拆成三个可以独立演化的资产：model weights、execution harness、operational skills。

对工程团队而言，skill 应该拥有类似代码的 provenance：source commit、evidence、verification test、failure case、dependency、适用范围和 benchmark delta。Skill creation 可以离线并行，验证任务可以隔离运行，最终 artifact 进入 versioned registry。在线 agent 只需要选择并加载相关分支。

一个很值得做的小实验，是为 Ray、vLLM、SFT 和 GRPO 分别构建 verified skill graph，然后在真实 research task 上记录：time-to-first-working-run、失败 tool call、重复搜索次数、最终任务质量与总 token cost。这样可以判断 skill library 带来的收益究竟发生在 setup、debug、optimization，还是 recovery。

## 另外两篇

第二篇是 **Post-Training Language Models for Gold-Medal Performance in Coding Competitions**。

它使用 22,000 道题、synthetic reasoning trace、SFT、RL 和 GenCorrect test-time refinement 训练 competitive-programming model。30B-A3B 的 Nemotron-3-Nano-CC 在 IOI 2025 上从 130 分提升到 291 分，再通过 GenCorrect 达到 468 分，超过 438.3 的金牌线。比赛专用的 Ultra-CC 系统在 IOI 2026 的同等时间、网络和提交限制下得到 535.4/600，高于最高人类选手的 498.27。

第三篇是 **HarnessDev: Can LLMs Create and Evolve Their Own Agent Harness?**。

HarnessDev 让模型创建并迭代 runnable harness，再用 held-out task success 和 execution-token cost 评估生成的 artifact。六个 creator model、四个领域、五个 benchmark 和 2,207 个 downstream instance 显示，自动生成的 harness 在 coding、search 和 research 上仍落后于成熟人工系统。Evolution 偶尔有效，但 hidden-task transfer 和跨 runtime-model transfer 都不稳定。

论文：<https://arxiv.org/abs/2609.02749>

另外两篇：<https://arxiv.org/abs/2609.02849>、<https://arxiv.org/abs/2609.01437>
