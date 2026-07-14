---
title: "Paper Digest: 2026-07-14"
categories: [Paper Digest]
tags: [AI, Post-training, Reinforcement Learning, Coding Agents, MCP, Software Engineering]
---

今天最值得看的 paper，我会选 **Weak-to-Strong Generalization via Direct On-Policy Distillation**。

它讨论的是一个很现实的 post-training 问题：RLVR 很有用，但在强模型上反复跑 RL 太贵。

现在很多 reasoning model 的提升来自 reinforcement learning with verifiable rewards。问题是，模型越强，每次 rollout 的成本越高。如果每一个新 strong model 都要自己生成大量 rollout，再用稀疏 reward 训练，post-training 本身会变成瓶颈。

Direct-OPD 的想法是：先在较弱模型上跑 RL，因为它便宜；然后把这次 RL 学到的改进信号迁移给更强的模型。

关键细节在于，它没有直接蒸馏 RL 后的小模型。

直接模仿小模型会混进小模型自己的能力上限。Direct-OPD 看的是同一个弱模型在 RL 前后的变化：post-RL teacher 相对 pre-RL reference，哪些 token、哪些动作变得更可能，哪些变得不那么可能。这个 log-ratio 被当成一种 dense implicit reward。

换句话说，它迁移的是 **policy shift**。

强模型仍然在自己的 on-policy states 上采样，但训练信号来自弱模型 RL 前后的相对变化。这样可以复用弱模型探索出来的方向，同时避免让强模型照着弱模型的最终策略走。

论文里的结果挺直接。

Direct-OPD 把 Qwen3-1.7B 在 AIME 2024 上从 **48.3%** 提升到 **58.3%**，只用了 **8 张 A100 训练 4 小时**。它还超过了 step-matched direct RL，并且可以把多个 policy shifts 顺序组合起来。

我觉得这篇最值得看的点，是它把 RL 的结果变成了可复用信号。

过去我们常把 post-RL model 当成产物：训练完一个 teacher，再让 student imitate。Direct-OPD 更像是在问：RL 到底改变了什么？能不能把这个变化本身拿出来，作为一种跨模型、跨规模的监督？

这个问题非常值得跟。

因为真正贵的部分往往是 exploration。让 strong model 自己探索当然直接，但成本很高。如果较弱模型能便宜地探索出方向，强模型只需要吸收这些方向，post-training pipeline 就会轻很多。

今天第二篇是 **When Does Restricting a Coding Agent to execute_code Help?**

它问的是 coding agent 里一个很工程的问题：agent 到底需要多少工具？

现在的 coding agents 往往暴露很多 tool surfaces：IDE primitives、bash、MCP code execution。不同系统对这件事有不同主张。有人认为工具越丰富越好，也有人认为只给一个 execute_code 会更稳定、更便宜。

这篇 paper 做了一个三组 ablation：baseline、bash_only、code_only。它同时测试 Claude Code 和 OpenAI Codex CLI，在 synthetic computation tasks 和 SWE-bench Mini modification tasks 上固定 model、harness、prompt。

结果有意思。

在四个 regime-agent 组合里，code_only 在三个组合中比最便宜的 tool-rich rival 更便宜，或者统计上打平；pass rate 在每个组合里基本打平。唯一例外是 SWE-bench 上的 Claude，code_only 方向上更贵，但差距主要来自失败轨迹上的成本，成功编辑本身没有明显变贵。

这个结果对 agent runtime 设计很实用。

很多时候我们讨论 tool design，容易只看能力边界：agent 能不能做某件事。但这篇把成本拉进来了。工具面越大，agent 的 action space 也越大，cache、失败恢复、无效探索都会变复杂。最后最便宜的工具面，取决于任务类型和 agent design 的组合。

第三篇是 **AgentCheck: A Reproduce-Intervene-Mitigate Workbench for LLM Agents over MCP**。

它关注的是 tool-using agent 的部署可靠性。

很多 eval 默认工具是对的：API 会返回正确结果，工具描述没被污染，数据是新的，timeout 只是偶发。但真实环境里，tool 可能返回一周前的值，可能超时，可能 description 被污染。更麻烦的是，agent 经常不会显式崩溃，而是自信地拿着错误工具输出继续推理。

AgentCheck 把 MCP server 变成一个可注入故障的测试面。

它先记录 agent 对真实工具的一次运行，然后在重放时注入 12 类故障。匹配的 tool calls 从 cache replay，agent 一旦偏离，后续调用再走 live tools。开发者可以打开一个 mitigation，再用同一个故障重跑，确认问题有没有消失。

论文报告五个 agents 里，最强的通过 105/120 个场景，最弱的只有 77/120。timeout 类问题可以被 retry mitigation 大幅改善，但 stale-data 类问题依然很难处理。

这篇对 MCP 和 personal agent 很有启发。

真正的 agent 质量，既包括模型会不会用工具，也包括工具坏掉时它会不会发现、会不会降级、会不会继续编一个看似合理的答案。

今天这三篇放在一起，主题很清楚：

**post-training 和 agent runtime 都在变得更模块化。**

Direct-OPD 把 RL 探索和强模型对齐拆开。execute_code ablation 把工具面当成可优化变量。AgentCheck 把工具故障变成可复现、可干预、可验证的测试场景。

如果你关心 RLVR、weak-to-strong post-training、coding agents、MCP runtime，今天先读 `2607.05394`。

## 今日 3 篇精选

### 1) Weak-to-Strong Generalization via Direct On-Policy Distillation
- 链接: https://arxiv.org/abs/2607.05394
- 摘要速读: 在弱模型上跑 RL，比较弱模型 RL 前后的 policy shift，再把这个变化作为 dense implicit reward 迁移给强模型。
- 为什么重要: 它试图降低 strong model RL exploration 的成本，并把 RL 学到的方向做成可复用、可组合的 post-training 信号。

### 2) When Does Restricting a Coding Agent to execute_code Help?
- 链接: https://arxiv.org/abs/2607.10569
- 摘要速读: 对 Claude Code 和 OpenAI Codex CLI 做 tool-surface ablation，比较 baseline、bash_only 和 code_only 在不同任务 regime 下的 pass rate 与成本。
- 为什么重要: coding agent 的工具面设计不能只看能力，还要看 cache-adjusted cost、失败轨迹成本和任务类型。

### 3) AgentCheck: A Reproduce-Intervene-Mitigate Workbench for LLM Agents over MCP
- 链接: https://arxiv.org/abs/2607.11098
- 摘要速读: 为 MCP tool-use agents 构建故障注入和重放 workbench，测试 timeout、stale data、poisoned description 等工具失败场景。
- 为什么重要: 真实 agent 部署里，最危险的失败往往是 agent 自信地使用错误工具输出。AgentCheck 让这些问题可以复现和验证。

## 一句话结论

今天最强的新信号是：**post-training 的探索信号、coding agent 的工具面、MCP agent 的故障处理，都可以被拆成更小、更可复用的工程模块。** `2607.05394` 值得先读。
