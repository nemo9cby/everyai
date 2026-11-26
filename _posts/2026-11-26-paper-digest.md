---
title: "Paper Digest: 2026-11-26"
categories: [Paper Digest]
tags: [AI, Agent RL, Coding Agents, GRPO, SWE-bench]
---

今天最值得看的 paper 是 **When Every Rollout Fails: Splicing Learner-Conditioned Options into Long-Horizon Agent RL**。

长程 agent 做强化学习时，有一个很现实的死局：同一组 rollouts 全部失败，reward 全是 0。GRPO 一类 group-relative 方法依赖组内 reward 差异计算 advantage，这时没有可用的 reward gradient。任务越长、terminal reward 越稀疏，这种 all-fail group 越常见。

SPLICE 的思路很直接。它保留 learner 自己走过的失败轨迹，只在某个关键决策点接入强模型，生成几个短而局部的候选动作。随后让当前 policy 从这些候选动作继续执行，用多次 continuation 的结果估计候选价值。只要某个局部 option 能带来正回报，它就可以成为训练目标。

这让一次完整失败的 rollout 仍然产生可学习信号，同时把强模型的参与限制在局部片段内。

## 为什么 all-fail group 会卡住 GRPO

Group-relative policy optimization 通常对同一个问题采样多条轨迹，再根据组内 reward 的相对高低更新 policy。若一组 reward 是 `[0, 0, 0, 0]`，标准化后的 advantage 也全部接近 0。模型付出了完整环境交互和 token generation 的成本，optimizer 却得不到区分好坏的信号。

常见补救方式是让更强的 teacher 提供完整成功轨迹。这个方案有两个代价：

- 完整轨迹的 proposal cost 很高，尤其是软件工程和 deep research 这类长任务。
- learner 会在很长的 teacher-generated history 上训练，off-policy 距离随着轨迹长度累积。

SPLICE 把 teacher assistance 缩短成一个 bounded option。失败轨迹的前缀仍然来自当前 policy，后续价值仍然由当前 policy 的 continuation 估计，训练分布因此更贴近 learner 真正访问的状态。

## SPLICE 的四个步骤

对于一个 all-fail group，流程可以拆成四步：

1. **选择 splice point**：proposal model 在失败轨迹里找到一个值得修复的决策点。
2. **生成局部 options**：从该位置提出多个长度受限的候选操作。
3. **估计 option value**：当前 policy 从每个 option 后继续执行多次，用终局结果估计局部选择的价值。
4. **构造训练目标**：把有正价值的 options 转成 return-weighted local targets，拟合当前 policy。

这里最关键的设计是 learner-conditioned。训练样本保留了 learner 自己造成的历史状态，修复目标也只覆盖一小段。模型学到的是如何从自己常见的失败前缀里做出更好的下一步。

## 实验结果

作者在两个 policy families、两个长程 agent tasks 上测试 SPLICE：

- Software Engineering，使用 SWE-bench Verified。
- Deep Research，使用 BrowseComp-Plus。

四个设置里，SPLICE 全部超过 GRPO 和 full-trajectory guidance baselines，相比 GRPO 提升 **2.8 到 9.8 个百分点**。

在初始阶段无法解决的 SWE tasks 上，SPLICE 将 GRPO 的 graduation rate 提高 **10.8 个百分点**。论文还报告，局部 option 的做法降低了昂贵 proposal-model computation。

这个结果说明，全失败样本仍然有训练价值。关键在于找到局部可干预点，并用 learner 自己的后续执行检验该干预是否真的改善终局结果。

## 对 agent post-training 系统的启发

SPLICE 同时提出了几个很适合系统优化的 knobs：

- **All-fail detection**：reward service 可以在组级别识别无梯度样本，并把它们路由到 splice queue。
- **Proposal concurrency**：proposal model 只负责短 options，可以与常规 rollout worker 分离调度。
- **Continuation fan-out**：每个 option 的重复 continuation 数量控制价值估计方差与环境成本。
- **Option length**：短 option 减少 off-policy 暴露，也限制强模型 token cost。
- **Prefix reuse**：失败 rollout 的环境状态和轨迹前缀可以复用，避免从头执行。

在 heterogeneous distributed RL stack 里，这些工作负载可以拆成不同资源池。强模型生成局部 proposal，廉价 learner 执行 continuation，sandbox workers 复用 checkpointed state，trainer 消费带 return weight 的局部样本。吞吐瓶颈会落在环境状态复用和 continuation 调度上，值得和算法指标一起测量。

## 今天另外两篇相关论文

**CARE-OPD** 研究长程 agent 的 on-policy distillation。它用 student-seeded teacher completions 覆盖 learner 真正访问的 histories，再通过 confidence gating 过滤不可靠更新，并根据 downstream execution evidence 决定哪些 prefix 值得更多 replay。在 SWE-bench Verified 上，它达到 59.5% success，同时比 asynchronous online OPD 减少 45% 训练时间和 44% environment actions。

**Negative-Only Policy Optimization** 关注 one-sided verifiability。很多廉价 verifier 能可靠指出失败，却无法证明通过检查的答案完全正确。NOPO 把未被否定的结果视为 unlabeled，只压低可信 negatives 的概率，在 code、logic 和 math 的多种 verifier 设置中改善 pass@k。

三篇论文指向同一个工程问题：长程 agent post-training 里的高价值信号很稀缺。失败轨迹、执行证据和单边可靠 verifier 都可以提供局部监督，训练系统需要把计算投入到真正能改变 policy 的位置。

论文：<https://arxiv.org/abs/2611.34777>

另外两篇：<https://arxiv.org/abs/2611.34756>、<https://arxiv.org/abs/2611.17287>
