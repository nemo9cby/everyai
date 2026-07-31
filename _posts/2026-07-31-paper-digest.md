---
title: "Paper Digest: 2026-07-31"
categories: [Paper Digest]
tags: [AI, GUI Agents, Reinforcement Learning, Post-Training, Code Agents, Computer Use]
---

今天最值得看的 paper，我会选 **Qwen-UI-Agent Technical Report: Toward Next-Generation Real-World Centric Foundation GUI Agents**。

它最有价值的部分是完整系统设计。模型、环境、online RL、action space、data flywheel 和 harness 被放在同一个闭环里讨论。对于真正要训练 agent 的团队，这些组件之间的接口往往比某一个 benchmark 数字更重要。

Qwen-UI-Agent 覆盖 mobile、computer use、browser 和 DeepSearch。它可以在同一条 trajectory 里交替使用 GUI、bash CLI 与 API，也支持一次 model turn 输出一组可并行或连续执行的 actions。在 computer-use 任务中，超过 40% 的 action outputs 采用 batching，trajectory 因而更短。

## 真实设备与大规模 sandbox

Mobile agent 最大的问题之一是 simulation-to-real gap。

模拟器里的 app 版本、权限弹窗、网络状态和 UI layout 相对可控。真实手机会出现通知打断、系统权限、登录状态、动态内容，以及同一应用不同版本之间的细节差异。一个在 simulator 上表现稳定的 policy，落到真实设备后经常因为这些小变化持续累积错误。

Qwen-UI-Agent 使用超过 100 台 physical mobile devices，覆盖 150 多个 applications。这套 real-device runtime 同时用于 task design、trajectory collection、online RL 和 evaluation。高风险操作还可以切换到 user takeover。

规模化 rollout 则主要由约 10,000 个 concurrent simulated environments 承担。真实设备提供分布保真度，sandbox 提供并发、重置能力和训练吞吐。二者共同解决 agent RL 中最棘手的环境供给问题。

## GUI 与 CLI 放进同一个 action space

GUI 的优势是覆盖面。只要人能够通过屏幕操作某个应用，agent 理论上也可以接入。

CLI 适合结构化任务，例如批量文件处理、文本变换、数据筛选和系统检查。让 agent 为所有步骤都点击界面，会增加 trajectory length，也会引入更多视觉定位和状态漂移。

Qwen-UI-Agent 的 unified action space 允许 policy 根据任务选择 GUI、CLI 或 API，并在同一次输出中组合多个 actions。这个设计的意义很具体：agent 可以在 GUI 中取得必要上下文，随后用 CLI 处理数据，再回到 GUI 提交结果。模型学习的是完整 execution policy，不需要人为规定每个阶段必须使用哪一种工具。

对于 OpenClaw、Claude Code 一类 workspace agent，这个抽象也很自然。屏幕、shell、browser 和 external tools 都是环境中的 action channels。训练目标需要同时覆盖工具选择、参数生成、执行后的状态理解，以及失败恢复。

## 超过 100 turn 的 online RL

短任务可以靠局部能力拿分。长任务会暴露 planning、state tracking、intermediate verification 和 recovery 的问题。

Qwen-UI-Agent 把 verifier-guided online RL 扩展到超过 100 interaction steps 的 trajectories。论文还使用 model-adaptive curriculum，优先采样成功率处于中间区间的任务。已经掌握的任务会被更难的任务替换，完全做不到的任务也不会持续占据 rollout budget。

这个 curriculum 对 environment-heavy RL 很重要。一次失败 trajectory 可能消耗上百次环境交互，如果 task distribution 长期包含大量过难样本，GPU、device 和 simulator 吞吐都会花在低信息量 rollouts 上。成功率中等的任务通常能产生更密集的 learning signal，也更容易观察 policy update 前后的变化。

从系统角度看，10,000 个并发环境只是起点。真正困难的是 reset semantics、environment health、trajectory storage、verifier latency、straggler handling，以及 rollout workers 与 learner 之间的数据节奏。论文展示的规模说明 GUI-agent post-training 已经具备典型 distributed RL system 的复杂度。

## AutoResearch-style data flywheel

Agent training 的数据瓶颈不只在数量，也在每轮训练后该补什么。

Qwen-UI-Agent 让 agents 参与 candidate task generation、task assessment、rollout processing、failure diagnosis 和 next-iteration planning。Human 主要承担 supervision 与 targeted revision。

这个闭环把 evaluation failure 直接连接到下一轮 data generation。假设模型在某类 permission dialog 上反复失败，系统可以聚合对应 trajectories、归纳 failure pattern、生成覆盖不同 app 和 device state 的新任务，再把它们放回 curriculum。

自动化分析仍然需要可靠的 quality gate。Agent 生成的任务可能缺少可验证目标，failure diagnosis 也可能把环境故障误判为 model error。数据飞轮的上限取决于 evaluator、task validator 与 provenance tracking 是否足够扎实。

## Harness 负责跨平台状态与主动服务

论文给出的典型场景是一条航班取消通知。

Harness 可以识别受影响的 itinerary，调用 DeepSearch 或 API 寻找替代方案，检查 calendar，在得到用户确认后通过 mobile GUI 完成改签，再到 desktop 环境更新会议安排并发送新的材料。

这里的核心是 state continuity。单个平台上的 task completion 只覆盖一段局部工作。真实用户目标经常横跨手机、电脑、网页、文件和消息系统。Harness 保存用户上下文、任务状态与审批边界，让 model 的多域能力组合成一条连贯 workflow。

Proactive service 带来的安全要求也更高。系统需要明确区分发现机会、提出建议、请求确认和执行外部动作。论文提到 user takeover，这一层仍值得看更多细节，包括权限模型、审计记录、撤销机制和长期记忆的使用边界。

## 结果怎样看

Qwen-UI-Agent 报告的主要结果包括：

- MobileWorld-Real：92.2%
- AndroidDaily：97.5%
- MobileWorld：82.1%
- OSWorld-Verified：79.5%
- OSWorld-v2 partial progress：40.0%
- WebArena：73.6%
- ScreenSpot-Pro：81.5%

MobileWorld-Real 包含 400 多个 tasks、100 多个 apps，Qwen-UI-Agent 比论文比较的几种 frontier models 高 3.5 到 7.5 percentage points。WebArena 上达到 73.6%，OSWorld-Verified 上排名第二。

这些数字说明系统具备很强竞争力，也要结合 evaluation setup 阅读。真实设备 benchmark 的 app state、账号、地区、任务可重复性和 harness 权限都会影响结果。OSWorld-v2 的 binary success 仍只有 13.9%，而 partial-progress 达到 40.0%，说明复杂 desktop tasks 里依然存在明显的最后一段完成度问题。

## 我最关心的三个问题

第一，long-horizon verifier 如何分配 credit。

超过 100 turn 的 trajectory 只使用终局 reward 会非常稀疏。论文值得进一步确认 intermediate verification、partial progress 与 failure classification 怎样进入 RL objective，以及这些信号是否会诱导 reward hacking。

第二，real-device experience 能否迁移到开放分布。

100 多台手机已经是很大的工程投入，但真实世界的 device、locale、accessibility settings 和 app versions 更复杂。需要观察模型对未见 UI、动态弹窗和登录异常的恢复能力。

第三，GUI+CLI policy 如何建立安全边界。

CLI 明显提高效率，也扩大了动作影响范围。Production harness 需要细粒度 permissions、sandbox、approval policy 和完整 audit trail。Agent 何时可以自由执行，何时必须等待确认，会直接决定这套系统能否进入长期运行的 personal assistant。

## 为什么今天选它

Qwen-UI-Agent 给出了一张相当完整的 agent training system 图纸：真实设备校准分布，大规模 sandbox 负责 rollout，online RL 训练长轨迹，adaptive curriculum 管理任务难度，GUI+CLI action space 提高执行效率，data flywheel 根据失败继续补数据，harness 连接跨平台状态与用户审批。

对做 post-training 和 distributed systems 的团队，它提供了很多可以独立验证的研究问题：environment concurrency 怎样换取有效样本，长轨迹 verifier 如何设计，curriculum 怎样控制 rollout 浪费，action batching 对 credit assignment 有什么影响，以及 real-device data 应该占多大比例。

## 另外两篇

第二篇是 **Frontis-MA1: Training an AI4AI Model towards Recursive Self-Improvement in Machine Learning Engineering**。

作者发布 OpenMLE full-stack system，并 post-train 一个 35B meta-evolution agent。训练与 inference 都围绕 Draft、Improve、Debug、Crossover 四种 atomic operators 展开。SFT 和 RL 使用 execution feedback 学习这些 operators，OpenMLE-Evo 再把它们组合成长时间 search。

在单张 RTX 4090、12 GB VRAM、每个任务 12 小时 budget 下，Frontis-MA1 加 OpenMLE-Evo 将 MLE-Bench Lite Medal Average 从 base model 的 39.39% 提高到 60.61%。加入 experience priors 与 asynchronous search 后达到 71.21%。这篇很适合关注 code RL、AI4AI 与 inference-time search 的读者。

第三篇是 **Change2Task: From Repository Changes to Executable Coding Agent Tasks and Environments**。

它从 merged pull requests 和 repository history 中构造 coding-agent tasks，并把历史变更迁移到健康的 modern repository revisions。三种重建方式分别是 Patch Reversal、Code Mapping 和 Agent Reconstruction。

在 1,130 个 eligible source changes 上，Change2Task 的 verified construction success 达到 79.6%，覆盖 bug fix、feature addition、test generation、API migration 和 security repair。对于需要持续扩充 SWE agent SFT、RL 或 evaluation data 的团队，这是一套很实用的 task factory 设计。

论文：<https://arxiv.org/abs/2607.28227>

另外两篇：<https://arxiv.org/abs/2607.28568>、<https://arxiv.org/abs/2607.28591>
