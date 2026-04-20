---
title: "Paper Digest: 2026-04-20"
categories: [Paper Digest]
tags: [AI, Post-Training, Reasoning, LLM]
---

今天最值得看的 paper，不是又一个更高分的 benchmark 结果，而是一篇把一个越来越现实的问题捅破的工作。**Where does output diversity collapse in post-training?** 关心的不是模型会不会答对第一条，而是模型在后训练之后，为什么越来越容易给出“整齐划一”的答案。

这件事乍看像个审美问题，其实很技术，也很现实。很多人默认，只要 top-1 更强，模型整体就更强。但很多推理系统、agent 系统、test-time scaling 方法，吃的不是 top-1，而是一个足够丰富的 candidate pool。你先要有多条不同但可能正确的路径，后面的 verifier、reranker、search、tree expansion 才有东西可选。如果后训练把这个空间提前压扁了，那么你在 inference 端加采样、加投票、加 rerank，很多时候只是从更窄的分布里反复捞同一种答案。

这篇 paper 做得好的地方，在于它没有把“多样性塌缩”简单甩锅给某一种后训练方法。作者沿着 Olmo 3 的三条后训练 lineage 去追，Think、Instruct、RL-Zero，跨 15 个任务、4 个 diversity metrics 去看，最后发现真正更关键的变量，是 **training data composition**。也就是说，问题很多时候不是你有没有在 inference 时显示 chain-of-thought，也不只是 SFT 还是 DPO，而是你拿什么数据、用什么分布，把模型往哪个更窄的行为盆地里推。

更有意思的是，作者还把 diversity loss 拆成两部分。一部分是“把错误答案洗掉了”，这当然有价值。另一部分则是“正确答案之间也变得越来越像”，这才是更值得警惕的地方。因为一旦正确候选之间的差异也被削平，很多依赖 pass@k、依赖多样候选探索、依赖 creative branching 的系统能力，都会一起被磨平。

## 今日 3 篇精选

### 1) Where does output diversity collapse in post-training?
- 链接: https://arxiv.org/abs/2604.16027
- 摘要速读: 作者系统分析后训练阶段到底在哪一步压缩了输出多样性，并把 Think、Instruct、RL-Zero 三条 lineage 放在一起比较。结论很关键，collapse 主要由训练数据组成决定，不能指望只靠 inference 端补救。
- 为什么重要: 这会直接影响 pass@k、test-time scaling、agent search，还有模型在开放任务里的探索空间。很多时候 benchmark 上更强的模型，在工作流里反而更“死板”，这篇 paper 给了一个更像样的解释。

### 2) Cut Your Losses! Learning to Prune Paths Early for Efficient Parallel Reasoning
- 链接: https://arxiv.org/abs/2604.16029
- 摘要速读: STOP 研究的是 parallel reasoning 里另一个非常贵的现实问题，坏分支能不能早点死。作者提出 path pruning taxonomy，并给出一个 learnable internal pruning 方法，在固定计算预算下提升 reasoning 表现。
- 为什么重要: 这篇 paper 说明 test-time scaling 这件事，重点可能已经从“多生成”转向“早止损”。系统强不强，越来越取决于它会不会及时砍掉没希望的路径。

### 3) Qwen3.5-Omni Technical Report
- 链接: https://arxiv.org/abs/2604.15804
- 摘要速读: Qwen3.5-Omni 是新的多模态技术报告，强调 hundreds-of-billions 参数、256k context、长时音频视频理解、Hybrid Attention MoE，以及更稳定的流式语音生成。
- 为什么重要: 它不是今天最锋利的一篇，但很能说明 frontier 模型的系统演化方向，多模态、长上下文、交互式输出、以及越来越像一个可以执行任务的统一模型栈。

## 一句话结论
今天最强的研究信号是，模型质量正在从“更会答第一条”转向“怎样保住有用的搜索空间，同时别把计算烧死”。`2604.16027` 值得重点看，因为它提醒人，后训练的代价不只是风格变化，也可能是把未来所有 test-time scaling 能利用的候选空间，一起悄悄压窄了。