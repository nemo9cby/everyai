---
title: "Paper Digest: 2026-05-29"
categories: [Paper Digest]
tags: [AI, Coding Agents, Post-Training, Software Engineering, Reinforcement Learning]
---

今天最值得看的 paper，我会给 **LiteCoder-Terminal: Scaling Long-Horizon Terminal Environments for Learning Language Agents**。

这篇的核心问题很实际：terminal agent 要学会多步规划、根据命令反馈修正动作、在动态环境里更新状态。但现在训练这类 agent 很依赖抓取来的外部代码仓库，domain 不够可控，验证信号不稳定，也很难专门针对某个能力短板造数据。

作者提出 **LiteCoder-Terminal-Gen**，一个 zero-dependency 的合成管线：从 domain specification 出发，自动生成可执行、可验证的 terminal training environments。

基于这个管线，他们做了两类资源：

- **LiteCoder-Terminal-SFT**：11,255 条 expert trajectories，覆盖 10 个 domain
- **LiteCoder-Terminal-RL**：602 个 verifiable environments，用于 trajectory-level preference optimization

训练结果也足够有信号。Qwen-family 模型做 SFT 后明显强于 base。32B 版本在 Terminal Bench 1.0、2.0 和 Pro 上分别达到 29.06%、18.54%、34.00% pass@1。之后再用 **Direct Multi-turn Preference Optimization (DMPO)**，还能继续提升。

这篇值得关注的地方在于，它把 terminal agent 的训练数据问题变成了一个可工程化的数据工厂问题。

对 coding agent 来说，环境比单条 prompt 更重要。真实任务里，模型要执行命令、读 stdout/stderr、修改文件、重新测试、处理状态变化。只用静态题目很难覆盖这种 interaction loop。LiteCoder-Terminal 的价值在于：环境本身是生成出来的，动作结果是可执行的，成功标准是可验证的，后续可以自然接 SFT 和 preference optimization。

这也是为什么我觉得它比今天更高热度的几篇通用 agent paper 更适合先读。它离 post-training system design 很近：怎么造环境，怎么收 expert trajectory，怎么做 verifiable reward，怎么把多轮执行轨迹用于 preference optimization。

今天另外两篇也值得一起看。

**How Coding Agents Fail Their Users** 做了 20,574 个真实 coding-agent sessions 的观察研究，覆盖 1,639 个 repo 和 IDE/CLI 两种 workflow。它把 misalignment 定义为开发者可见的 pushback，并从 form、cause、cost、resolution 四个轴标注。结论很有产品味：大多数失败没有造成不可逆破坏，90.50% 主要带来 effort 和 trust cost，但 91.49% 的可见解决仍然需要用户明确纠正。这个 taxonomy 很适合拿来做 coding-agent eval 和 telemetry labels。

**RADAR at Meta** 则是生产部署视角。Meta 用 risk-aware funnel 自动 review 低风险 diff：先按 authorship/source type 分类，再过 eligibility gates、static heuristics、Diff Risk Score、LLM automated code review 和 deterministic validation。论文覆盖 535K+ reviewed diffs，其中 331K+ landed。它的启发是：AI 带来的代码吞吐增长会把 code review 变成系统瓶颈，而解法需要风险校准和多层验证。

三篇放在一起看，今天的信号很清楚：coding agent 正在进入更 operational 的阶段。训练侧需要可执行环境，评测侧需要真实失败 taxonomy，部署侧需要 risk-calibrated automation。

所以今天先读 `2605.29559`。如果你关心 coding agent、SFT/RL、Terminal-Bench、或者想设计更靠谱的 agent post-training pipeline，这篇最值得认真拆。

## 今日 3 篇精选

### 1) LiteCoder-Terminal: Scaling Long-Horizon Terminal Environments for Learning Language Agents
- 链接: https://arxiv.org/abs/2605.29559
- 摘要速读: 用合成管线生成可执行、可验证的 terminal training environments，并构建 SFT trajectories 与 RL preference environments。
- 为什么重要: 它把 coding/terminal agent 的训练数据、环境验证和多轮偏好优化连成了一条可复用 pipeline。

### 2) How Coding Agents Fail Their Users
- 链接: https://arxiv.org/abs/2605.29442
- 摘要速读: 分析 20,574 个真实 coding-agent sessions，整理 developer-agent misalignment 的形式、原因、成本和解决方式。
- 为什么重要: 它给 coding-agent evaluation 提供了真实 failure taxonomy，不只看 benchmark pass rate。

### 3) Automating Low-Risk Code Review at Meta: RADAR, Risk Calibration, and Review Efficiency
- 链接: https://arxiv.org/abs/2605.30208
- 摘要速读: Meta 用风险分层、LLM review 和 deterministic validation 自动处理低风险 diff，覆盖 535K+ reviewed diffs。
- 为什么重要: 它展示了 AI code growth 之后，大型工程组织如何用 risk calibration 承接 review bottleneck。

## 一句话结论
今天最强的研究信号是：**coding agent 的关键基础设施正在补齐：可执行训练环境、真实失败 taxonomy、生产风险闸门。** `2605.29559` 值得先读。
