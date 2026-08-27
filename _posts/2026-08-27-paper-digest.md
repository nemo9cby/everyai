---
title: "Paper Digest: 2026-08-27"
categories: [Paper Digest]
tags: [AI, Agents, Agent Harness, Post-Training, Reinforcement Learning, SFT]
---

今天最值得看的 paper，我会选 **JIT-Agent: Scaling Harness Intelligence via Just-in-Time Harness Evolution**。

同一个 foundation model，放进不同的 agent runtime，表现可能差很多。Memory 怎么裁剪，任务怎么拆，tool 在什么时候暴露，失败之后如何恢复，这些选择共同构成 harness。它们决定模型看到什么、能做什么，以及一次长任务会不会在几十步之后偏离目标。

JIT-Agent 提出一个很大胆的方案：训练一个专门生成 harness 的 27B meta-agent。每次收到任务，它现场写出一套针对当前问题和当前 backbone 的可执行 harness，再把普通 agentic LLM 放进去运行。

论文把这种能力称为 **harness intelligence**。

## 一套 harness 被拆成四个模块

JIT-Agent 用固定协议把 harness 表示为四个可组合模块：

1. Memory，决定保留哪些历史与工作状态
2. Planning，形成下一步 directive，并在执行中更新计划
3. Action，管理 agent loop、验证和失败恢复
4. Capability，选择并编排 tools 与 skills

这个分解给 harness generation 加了一层明确边界。模型生成结构化、可验证的模块，输出随后经过 protocol validator、compiler check 和 runtime check。搜索型任务可以获得并行 evidence exploration，规划任务可以获得 constraint tracker，workspace 任务可以把文件、patch、trace 和 application state 纳入 working memory。

同一套固定 runtime 很难同时适配这些需求。JIT-Agent 把适配动作放到每个 task instance 上，生成结果也会参考 backbone、tool registry 和历史 harness archive。

## 三阶段训练：生成、修复、优化

JIT-Agent 的训练流程很像一条完整的 agent post-training pipeline。

Stage I 用强 teacher 为不同任务生成符合四模块协议的 harness。通过 validation 和 execution check 的样本进入 SFT。随后加入 preference learning，偏好 task reward 更高、latency 和 cost 更低的候选。

Stage II 专门学习 repair。Stage I 中编译失败、interface mismatch、tool-call failure 和 runtime exception 的样本会被保存，teacher 根据 diagnostic report 生成结构化 patch。论文只保留两轮之内能够修复的 trajectory，让模型学习 deployment 中最常见的局部恢复。

Stage III 使用 Evo-GDPO。每轮从当前 policy 采样一组 harness，与 archive 中的 incumbent 在相同 backbone、budget 和 evaluation seed 下执行。训练信号分成三条：

- task reward
- latency
- monetary cost

三个信号分别 normalization，再合成 advantage。Reward 保持主导，latency 和 cost 只有在候选不损失 incumbent reward 时才获得正向激励。通过这一约束，模型很难靠无限拉长 trajectory 换分。

被 archive 接受的候选也要满足 frontier 条件：reward 不低于当前最好结果，并且在 reward、latency 或 cost 至少一个维度严格改善。

## Controlled comparison 才是论文最有价值的结果

JIT-Agent 在九个 benchmark 上测试 deep research、daily work、planning 和 workspace tasks。

把默认 scaffold 换成 JIT-generated harness 后，18 个相同 backbone 与 benchmark 的配对全部提升：

- GLM-5.2 的九项平均分从 74.1 提升到 81.8
- DeepSeek-V4-Flash 从 66.7 提升到 75.5
- DeepPlanning-Shopping 上，DeepSeek-V4-Flash 从 59.1 提升到 83.9
- DeepPlanning-Travel 上，GLM-5.2 从 62.8 提升到 83.0

论文还固定 backbone，对比 Claude Code、Codex、OpenCode、Hermes 和 NanoBot。JIT-Agent 在六个设置中拿到四个最高 performance，并且在全部六个设置中使用最少 token、最低 API cost。

相对每项中最便宜的 fixed harness，它把单 case 成本平均降低 36.0%。在 DeepSeek-V4-Flash 的 xBench-DS 上，JIT-Agent 把 performance 从最强 fixed harness 的 78.0 提高到 82.0，同时把 token 从 527K 降到 212K，成本从 0.075 美元降到 0.039 美元。

这组 controlled result 很关键。更高分可以来自更长 context、更多 retries 或更多 tool calls。JIT-Agent 的 trajectory 通常更短，说明 task-conditioned harness 确实改变了执行资源的分配方式。

它也没有在所有 quality 指标上获胜。DeepSeek-V4-Flash 的 AgentIF 中，JIT-Agent 比 Claude Code 低 3.1 分；Qwen3.6-Flash 的 DeepSearchQA 中，它比 NanoBot 低 3.9 分。两项都换来明显更低的 token 和成本。这些例外让论文的 cost-performance frontier 更可信，也提醒团队根据 workload 选择 operating point。

## 对 agent infrastructure 的启发

训练 harness generator 会把一批系统对象纳入 post-training provenance：

- task 与 evaluator seed
- foundation model checkpoint
- tool 与 skill registry
- harness protocol 和生成代码
- compiler、runtime 与 verifier trace
- reward、latency、token 和 monetary cost
- 被检索的历史 harness 与 archive frontier

缺少这些版本信息，团队很难判断提升来自 backbone、harness、工具升级，还是 evaluator variance。

对 distributed rollout system，harness candidate 很适合成为独立调度单元。Worker 可以先做静态 validation，再执行少量 paired rollouts；失败样本进入 repair queue；通过 frontier gate 的候选进入 archive。Reward、latency 和 cost 必须使用同一组 seeds 和预算测量，否则 group-relative comparison 会被执行环境噪声污染。

更深一层的问题是联合优化。Harness 改变 policy 能观察的状态和可调用的动作，policy update 又会改变最优 harness。实践中可以交替固定一侧：固定 backbone 做 harness search，固定 harness 做 model post-training，最后在独立 task family 上做联合 held-out evaluation。

JIT-Agent 给出的核心判断很有力量：agent capability 属于 model 与 harness 的组合。模型权重可以训练，包在权重外面的 memory、planner、action loop 和 capability orchestration 同样可以被训练。

## 另外两篇

第二篇是 **D^3-MOPD: Adaptive Dynamic Domain ScheDuling for Efficient Multi-Teacher Distillation**。

它处理 multi-teacher on-policy distillation 中固定 domain mixture 的浪费。不同 domain 收敛速度差异很大，固定采样会继续给已经 plateau 的 domain 分配 rollout。D^3-MOPD 直接复用训练里已有的 per-domain reverse-KL，由异步 watcher 估计 headroom 与 improvement rate，再调整 domain sampling ratio。Qwen3.6-35B-A3B student 在四个 expert teacher 上恢复 97% 的平均能力差距，固定 mixture 只有 63%，并用约三分之一 rollout steps 达到相同 peak。

第三篇是 **Is Next-Chunk Reasoning RL Really Better than SFT? Revisiting Training Strategies under no-CoT Data**。

这篇做了一个很值得 post-training 团队警惕的 controlled comparison。面对包含推导过程、却没有显式 CoT 标注的数据，简单的 Mixed SFT 把 no-CoT 和 long-CoT 放进一个 supervised stage，最终 post-RLVR ceiling 高于 next-chunk reasoning RL，训练 compute 低 60 倍以上。论文还发现 pre-RLVR accuracy 更高，不保证 RLVR 之后更强。评估一种中间训练策略时，最终 pipeline quality 和总 GPU-hours 才是更可靠的指标。

论文：<https://arxiv.org/abs/2608.25593>

另外两篇：<https://arxiv.org/abs/2608.24987>、<https://arxiv.org/abs/2608.23256>
