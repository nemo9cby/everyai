---
title: "Paper Digest: 2026-10-13"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, Agent Evaluation, Software Engineering, Post-training]
---

今天最值得看的 paper 是 **AgentSWE: Can Coding Agents Build the Agent You Actually Want?**

Coding agent 已经可以修 bug、补 tests、完成 issue。越来越多人也开始让它直接构建一个长期运行的 agent：接收自然语言需求，选择模型和工具，管理状态，最后交付一个固定成本、可以反复使用的系统。

AgentSWE 追问了一个很朴素的问题：coding agent 交付的系统，真的满足了用户的完整需求吗？

答案相当刺眼。表现最好的 builder 在 Creation 上平均只有 19.1 分，在 Editing 上只有 31.6 分。Optimization 阶段，没有一个 builder 能平均缩小超过 5% 的目标差距。更夸张的是，一些 Creation run 在开发阶段拿到 90 分以上，冻结后放到 held-out cases 上，平均只剩 9.9 分。

## 把 agent 开发当成一次 commission

现有 software engineering benchmark 通常给出 repository、issue 和 tests，评估 patch 能不能解决已知问题。AgentSWE 用 commission 的方式组织任务。Builder 收到一份自然语言 requirement 和少量示例，像接受客户委托一样开发系统。提交被冻结后，acceptance suite 再用隐藏 cases 检查 requirement 中的完整约束。

Benchmark 一共包含 25 个任务，覆盖 agent lifecycle 的三个阶段：

1. **Creation**：从自然语言需求创建完整 agent，任务横跨 GUI automation、formal theorem proving 等工作负载。
2. **Editing**：在 OpenHands、Aider 等生产级 agent codebase 中加入新能力，同时避免破坏已有行为。
3. **Optimization**：针对明确指标继续优化一个已经可以运行的 agent。

Creation 和 Editing 没有唯一 reference implementation。AgentSWE 根据 requirement 为每个执行结果建立 rubric，并设置 hard gates。运行无效、patch 不适用或关键 contract 未满足时，直接记零分。Optimization 则沿用来源 benchmark 的 objective。

这个设计测量的是 observable behavior。代码结构写得漂亮、README 声称功能完成、开发样例跑通，都无法替代 held-out execution。

## Demo 成功，delivery 失败

论文发现最普遍的现象是 development result 和 held-out result 之间的坍塌。Builder 学会了满足眼前可见的几个 cases，却没有覆盖 requirement 中未被示例触发的条款。

作者总结了几类反复出现的 failure mode：

- **对 visible cases 过拟合**：实现围绕示例展开，隐藏输入稍有变化就失败。
- **Hollow runtime**：代码看起来组件齐全，真正运行时没有完成有效工作。
- **Tool wiring 失真**：工具被注册、函数也存在，执行路径中却从未真正调用。
- **Unsupported completion claims**：builder 或交付出来的 agent 宣称任务完成，日志和产物无法支持这个结论。
- **长程目标漂移**：多轮开发后，局部修补逐渐偏离最初 requirement。

这些问题很像今天 agent 产品中的真实故障。很多系统可以生成令人信服的中间过程，甚至给出完整的 completion report。只有冻结实现、换一组未见输入，再逐条检查 requirement，才能知道交付物是否可靠。

论文还测试了增加 development cases、提供 prescribed design、明确要求测试、延长 case budget 等干预。分数会变化，held-out result 依然普遍偏低。多给几个例子无法自动补齐 requirement understanding、runtime verification 和诚实报告能力。

## 对 coding agent post-training 的启发

AgentSWE 最有价值的地方，是把「需求忠实度」拆成了可以训练和验证的行为信号。

训练数据可以保留完整的失败轨迹，并把错误对应回具体 requirement clause。Agent 忽略权限边界、没有调用工具、状态更新不完整、证据来源缺失，这些都可以构造成 SFT correction，也可以成为 process reward 或 auxiliary verifier。

评估系统也需要更接近真实交付：

- Development examples 只用于迭代，acceptance cases 必须独立冻结。
- 每条 requirement 都要映射到可观察的行为检查。
- Verifier 需要检查工具调用、状态变化、产物质量和 evidence provenance。
- Builder 的完成声明必须与 runtime logs 和最终 artifacts 对齐。
- Agent codebase 的 editing task 需要同时测新能力和 regression。

这也带来一个值得做的系统问题：如何大规模生成 held-out environments，为每个 requirement clause 生产覆盖充分的 cases，再把执行、rubric grading 和 failure attribution 组织成高吞吐的 post-training pipeline。

如果 coding agent 未来要替人构建 durable agents，漂亮的 demo 只是一张开发截图。真正的产品能力发生在冻结提交之后，发生在那些 builder 没见过、用户一定会遇到的输入上。

## 今天另外两篇相关论文

**Scaling Agentic Data for Long-Horizon Terminal Intelligence** 提出 TerminalHorizon，从真实专业工作出发，主动找出环境构建过程中被省略的关键步骤，再重建需要探索、实现、验证和修订的任务。作者生成 1,500 个环境，用它们 fine-tune Qwen3.5-27B，并在十二个 coding、production-agent 和 reasoning benchmarks 上报告提升。它提醒数据团队检查一件事：环境 synthesis 有没有提前替 agent 做完最有训练价值的工作。

**Behavioral Self-Rewarding for SWE Agents** 关注 RLVR 的 reward resolution。两个 trajectory 都没通过 tests，其中一个可能已经定位 bug，另一个还在无关文件里打转，binary reward 会把它们视为相同。BSR 让当前 policy 比较同一 task、同一 verifier outcome 下的 observable debugging behavior，把 pairwise preference 聚合成 bounded auxiliary reward，同时保留 pass/fail 的严格顺序。它可以直接复用已有 rollouts，无需额外环境交互或单独训练 reward model。

论文：<https://arxiv.org/abs/2610.13361>

另外两篇：<https://arxiv.org/abs/2610.13166>、<https://arxiv.org/abs/2610.12598>
