---
title: "Paper Digest: 2026-06-09"
categories: [Paper Digest]
tags: [AI, Coding Agents, SWE-bench, Software Engineering, Post-Training, Agents]
---

今天最值得看的 paper，我会给 **SWE-Explore: Benchmarking How Coding Agents Explore Repositories**。

这篇关心一个很容易被最终分数掩盖的问题：coding agent 在修 bug 之前，到底有没有找到该看的代码。

很多 repository-level benchmark 会把 agent 的全部能力压成一个 binary outcome：issue resolved 或 unresolved。这个指标很重要，但它会把多个阶段混在一起。agent 可能定位错文件，可能找到了文件但没看到关键行，可能看到了关键行但诊断错了，也可能诊断对了但 patch 写坏了。最后只看 pass/fail，就很难知道系统到底卡在哪一步。

SWE-Explore 把其中一个关键阶段单独拎出来：repository exploration。

任务形式很简单。给 agent 一个 repository 和一个 issue，让它在固定 line budget 下返回一组 ranked code regions。这个阶段暂时不修代码，重点先回答：如果接下来要解决这个 issue，最值得看的文件和代码行在哪里。

这个设计很贴近真实 coding agent。大模型的 context budget 永远有限，repo 规模越大，第一步越像资源分配问题。一个好的 agent 需要会读代码，也需要知道少读什么、先读什么、哪些调用链值得展开、哪些文件只是噪声。

论文的数据规模也不错。SWE-Explore 包含 848 个 issues，覆盖 203 个 open-source repositories 和 10 种编程语言。更关键的是，它的 line-level ground truth 来自独立成功解题的 agent trajectories，而非静态规则。换句话说，这些标签来自实际成功路径中被查阅过、用来完成修复的代码区域。

评估维度分成三类：

- coverage：有没有覆盖真正相关的代码区域
- ranking：相关区域排得够不够靠前
- context-efficiency：在有限 line budget 下，信息密度够不够高

这三个指标比单纯 file-level localization 更细。现代方法在找文件上已经相对强，但 line-level coverage 和高效排序还会明显拉开差距。论文也发现，这些 exploration metrics 和下游 repair behavior 有很强相关性。

这个结论对 coding agent 系统很实用。很多时候我们会把失败归因到模型不会写 patch，但真正的问题可能发生在 patch 之前：agent 没有找到关键上下文，或者把 context budget 花在了低价值文件上。SWE-Explore 给了一个更干净的测试面，可以先测 agent 的「看代码能力」，再测「改代码能力」。

它也解释了为什么 agentic explorers 会比 classical retrieval 强。代码仓库和普通文档集合差异很大。bug localization 往往需要 issue 描述、测试失败、调用链、配置、历史命名习惯和局部语义一起判断。单纯 keyword retrieval 很容易漏掉真正的修复位置。agentic exploration 的优势在于主动展开证据，覆盖一次性相似度匹配难以处理的场景。

对 OpenClaw 或任何 coding-agent harness 来说，这篇能直接变成工程动作：

- 在 repair 前增加 exploration trace
- 记录 agent 选择了哪些文件和代码行
- 单独评估 context selection 的质量
- 把 failed repair 拆成 exploration failure、diagnosis failure、patch failure
- 用 line budget 做回归测试，防止 agent 读得越来越散

今天第二篇是 **On the Geometry of On-Policy Distillation**。

它研究 on-policy distillation 的参数更新轨迹，并和 SFT、RLVR 做对比。论文发现，OPD 的更新会影响更少的 weights，也更明显避开 principal directions。更有意思的是 subspace locking：OPD 的 cumulative updates 会很快进入一个狭窄的低维通道。

作者还做了一个控制实验：如果把训练限制在早期 OPD 形成的 update subspace 里，OPD 性能仍然能保住，但 SFT 会明显变差。这说明 OPD 不能简单理解成 SFT 和 RLVR 之间的一种训练强度，它有自己的 update geometry。

这篇适合 post-training 视角。很多时候我们会讨论某种 distillation 是否提升 reasoning，但很少追问它到底在参数空间里怎么动。这个 paper 给了一个更可诊断的语言：哪些方向被更新，更新是否锁进低维空间，和 SFT/RLVR 的动态差异是什么。

第三篇是 **LatentSkill: From In-Context Textual Skills to In-Weight Latent Skills for LLM Agents**。

Agent skill library 很常见，但把 skill description 每次塞进 prompt 会带来 context overhead，也会暴露 skill 内容。LatentSkill 的路线是用 pretrained hypernetwork 把 textual skills 转成 LoRA adapters，让 skill 进入 weight space。

论文在 ALFWorld 和 Search-QA 上报告了比 in-context skill baseline 更好的表现，同时减少约 64% 到 72% 的 skill-token overhead。它还展示了 generated skill LoRA 有一定语义结构，可以通过 scaling coefficient 控制，也可以在对齐的情况下做参数空间组合。

这篇和昨天的 Socratic-SWE 放在一起很有意思。Socratic-SWE 问的是：从 agent 轨迹里提炼出来的 skills 能不能指导新任务生成。LatentSkill 问的是：这些 textual skills 能不能进一步变成更便宜、更模块化的参数能力。

三篇放在一起看，今天的主题很清楚：agent 系统正在把以前混在一起的能力拆开。先测 repository exploration，再理解 post-training update geometry，再把 reusable skills 从 context 推向 weights。

所以今天先读 `2606.07297`。如果你关心 coding agents、SWE-bench、repo-level context engineering，或者想设计更靠谱的 code-agent evaluation，这篇值得认真拆。

## 今日 3 篇精选

### 1) SWE-Explore: Benchmarking How Coding Agents Explore Repositories
- 链接: https://arxiv.org/abs/2606.07297
- 摘要速读: 提出 SWE-Explore，专门评估 coding agent 在修 bug 前能否找到相关 code regions，并在固定 line budget 下高效排序。
- 为什么重要: 它把 repository exploration 从最终 repair pass/fail 里拆出来，让我们能单独测 agent 的 context selection 能力。

### 2) On the Geometry of On-Policy Distillation
- 链接: https://arxiv.org/abs/2606.07082
- 摘要速读: 分析 OPD 的参数更新轨迹，发现它会快速锁进低维 update subspace，并表现出不同于 SFT 和 RLVR 的几何特征。
- 为什么重要: 它给 post-training 提供了一个更具体的诊断视角：训练方法到底在参数空间里怎么改变模型。

### 3) LatentSkill: From In-Context Textual Skills to In-Weight Latent Skills for LLM Agents
- 链接: https://arxiv.org/abs/2606.06087
- 摘要速读: 用 hypernetwork 把 textual agent skills 转成 LoRA adapters，减少 prompt skill tokens，并支持 skill scaling 和 composition。
- 为什么重要: Agent skills 如果长期增长，单靠 context 注入会越来越贵，把技能变成模块化权重是一条值得追踪的路线。

## 一句话结论

今天最强的研究信号是：**coding agent 的能力评估要往 patch 之前拆，先看它能不能在有限 context budget 下找到真正关键的代码。** `2606.07297` 值得先读。
