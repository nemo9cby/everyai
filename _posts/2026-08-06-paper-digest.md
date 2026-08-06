---
title: "Paper Digest: 2026-08-06"
categories: [Paper Digest]
tags: [AI, Agents, Reinforcement Learning, Search Agents, Post-Training, Code Generation]
---

今天最值得看的 paper，我会选 **ABSeeker: Training Long-Horizon Search Agents via Answer-Backtracked Credit Assignment**。

它研究 long-horizon agent training 里最难处理的问题之一：一条搜索 trajectory 最终成功或失败，训练系统该怎样判断中间两百步里哪些动作真正有价值？

Trajectory-level reward 会把同一个 outcome 分给整条轨迹。成功轨迹中的错误搜索、无效绕路和侥幸猜测也会被强化；失败轨迹中已经找到的关键证据、正确排除的候选和合理的探索则会一起受到惩罚。ABSeeker 的核心方法 Answer-Backtracked Credit Assignment（ABC）利用 verified answer 反向恢复证据链，再让这些 evidence clues 成为每一步的评分锚点。

## 从答案反向恢复 clue chain

ABC 的第一阶段叫 Answer-Backtracked Clue Recovery。

输入是一道搜索问题和它的 verified answer。Recovery model 从答案出发，逐步寻找能够连接答案与问题约束的 entities、facts 和 relations，并用真实网页内容验证这些线索。最终得到的 clue set 描述了一条足以解释答案的 evidence chain。

这一步避开了 gold trajectory 的依赖。同一道复杂搜索题可以有很多条有效路线，强迫 student 模仿一条固定路线会丢掉 on-policy exploration 的价值。Clue set 只定义需要发现或验证的事实，agent 仍然可以用自己的顺序、query 和工具调用抵达这些事实。

论文给出的例子要求寻找一个满足多重约束的护肤品牌，正确答案是 CeraVe。Recovery model 从答案回溯出 ceramides、L'Oréal acquisition、创始人教育经历等六条线索。随后，无论 student 的完整 trajectory 成功或失败，每个 search step 都可以根据它是否发现、验证、错误放弃这些 clues 来评分。

## 把每一步变成可解释的 reward

Clue-Anchored Step Scoring 会同时看到当前 step、原始 query 和完整 clue set。这里的 step 包含 reasoning、tool call 与 tool response。Scorer 输出一个数值 reward 和简短 rationale。

每一步从 1.0 的 neutral score 开始：

- 发现或验证正确 clue：+0.8
- 排除错误候选：+0.4
- 错误否定正确 clue：-0.8
- 提交 verified answer：+1.0
- 提交错误答案：-1.0

同一步中的多项行为可以累加，最终分数被截断到 [0, 2.0]。因此，一个失败 trajectory 中发现关键证据的 step 仍可获得 1.8；一个成功 trajectory 中错误放弃正确线索的 step 仍会被降权。

作者分析 8.5K 条 SFT trajectories 后发现，成功轨迹中约有 4% 的 steps 得分低于 neutral 1.0，失败轨迹中接近 10% 的 steps 得分高于 1.0。这个分布说明 binary outcome 会系统性混淆局部动作质量，长链 agent 尤其严重。

## 同一套 step reward 用于 SFT 和 GRPO

ABC 把 step scores 用在两个连续训练阶段。

第一阶段 ABC-SFT 保留正确与错误 trajectories，并将每一步的 reward 映射成 sigmoid loss weight。论文实际采用 `w(r) = 2σ(2(r-1))`，neutral reward 对应权重 1.0。高分动作对梯度贡献更大，低分动作的 imitation signal 被压低。Tool responses 全部 mask，只优化 policy 自己生成的 tokens。

第二阶段 ABC-GRPO 从 SFT checkpoint 开始 online RL。每个问题采样一组 rollouts，step rewards 在 rollout group 内归一化，再用 discounted return 向较早步骤传播。论文使用 `γ = 0.25`，让未来 clue discovery 能给此前的搜索决策分配部分 credit。得到的 step-specific advantage 会共享给该 turn 中所有 policy tokens，最后进入标准 clipped GRPO objective。

这套设计有两个值得注意的地方。第一，SFT 数据没有丢弃 failed trajectories，其中仍可能包含高质量 exploration。第二，RL advantage 对齐 interaction step，训练粒度更接近 agent 在环境中采取的真实动作。

## 训练配置与结果

ABSeeker 以 Qwen3.5-4B 为 backbone。SFT 数据来自 OpenSeeker，共 8.5K trajectories，其中 5.5K 最终答对，3.0K 最终答错，最长 200 steps，训练 3 epochs。

RL 使用 1,000 道问题，每题 8 条 rollouts，每条最多 200 interaction turns。系统使用 veRL、16 个 asynchronous agent-loop workers、temperature 1.0 和 top-p 0.95。Clue recovery 与 step scoring 都由 DeepSeek-V4-Flash 完成。

不使用 context management 时，ABSeeker 在 BrowseComp 达到 37.3%，BrowseComp-ZH 达到 39.1%。对比标准 pipeline：

- Standard SFT：BrowseComp 28.5%，BrowseComp-ZH 30.4%
- ABC-SFT：30.8%，31.8%
- Standard GRPO：33.5%，36.3%
- ABC-GRPO：37.3%，39.1%

收益也延伸到 xbench 与 GAIA-text。最终模型在五个 benchmarks 上都超过同规模 4B search-agent baselines。

加入 256K context 和最多五轮 discard-all context management 后，BrowseComp 从 37.3% 增加到 55.3%，BrowseComp-ZH 从 39.1% 增加到 52.9%。这个结果也提醒我们，policy、training signal 和 agent harness 会共同决定最终表现。只看模型 checkpoint 很容易低估 context strategy 的贡献。

## 对 agentic post-training 的启发

最重要的想法是把 verified outcome 展开成 intermediate anchors。

很多 agent tasks 只有稀疏的终局信号，过程却包含可验证的 subgoals。Web search 可以从答案恢复 evidence clues；coding task 可以从 passing implementation 反向提取 required behaviors、invariants 和 test-relevant states；GUI task 可以从目标状态恢复必要的 intermediate state transitions。只要这些 anchors 允许多条有效路径，便能为 on-policy trajectory 提供更细的监督，同时保留 exploration diversity。

它也给 distributed rollout system 带来明确的工程代价。8.5K 条长 trajectories 需要先做 clue recovery，再逐 step 调用强模型评分。RL 阶段每个问题采样 8 条最长 200 turns 的 rollouts，同时运行 16 个 agent loops。离线 scoring 的 batching、judge cost、reward caching，以及 rollout 和训练之间的异步调度，会直接影响方法能否扩展到更大模型和更多任务。

另一个风险来自 judge bias。Recovered clues 可能遗漏一条同样有效的证据链，step scorer 也可能偏好特定搜索风格。论文中的 verified web evidence 与评分 rationale 提供了一部分可审计性，但更完整的研究仍需测量 clue recall、scorer consistency，以及不同 judge models 对 policy 的影响。

## 为什么今天选它

ABSeeker 给出了一条完整的 post-training recipe：verified answer 负责锚定真相，answer backtracking 负责恢复过程结构，clue scoring 负责生成 dense rewards，ABC-SFT 和 ABC-GRPO 再把同一套 credit 用于 imitation 与 exploration。

它对失败数据的处理尤其值得保留。长链 agent 的失败很少意味着此前每一步都没有价值。训练系统能够识别失败轨迹中已经做对的部分，rollout budget 才有机会沉淀成可复用的 policy improvement。

## 另外两篇

第二篇是 **ExeCRE: Execution-Consistency Guided Reliability Estimation for Self-Correcting Code Generation**。

ExeCRE 关注 code self-correction 中 feedback 的可信度。方法对大量随机输入执行多个 candidate programs，根据输出一致性构造 signals，再用 Dawid-Skene 模型推断 latent code reliability。这个估计用于判断一次 correction 是否值得执行。在 GPT-5.2 与 LiveCodeBench 上，已经正确的程序被误导修改的平均案例数从 representative baseline 的 113.2 降到 14.0。

它提供了一个很实用的 verifier 视角：当 reference implementation 和完备 tests 都缺失时，可以先从多个执行者的结构化 agreement 中估计 correctness，再决定是否相信 synthetic feedback。它也适合与 agent rollout、test generation 和 weak supervision 结合。

第三篇是 **SciCode-Verified: How Benchmark Defects Underestimated the Scientific-Coding Ability of Language Models**。

作者逐题审计 SciCode 的 65 道测试题，发现 263 个缺陷。其中 192 个缺陷分布在 91% 的 main problems 中，会通过无法复现的 gold answers、过紧 tolerance 或自相矛盾的 specification 错误拒绝正确答案。78% 的 suppressing defects 需要 physics 或 mathematics 专业知识才能发现。

修正后，十二个 frontier model snapshots 的 subproblem accuracy 从 45-60% 增加到 84-98%，main-problem accuracy 从 9-27% 增加到 69-92%。这篇 paper 对 post-training 的警告很直接：executable grader 也需要版本管理、domain-expert audit 和 disagreement analysis。错误 verifier 一旦进入 reward loop，影响会超出 leaderboard，直接改变模型学习到的行为。

论文：<https://arxiv.org/abs/2608.05102>

另外两篇：<https://arxiv.org/abs/2608.04439>、<https://arxiv.org/abs/2608.04975>
