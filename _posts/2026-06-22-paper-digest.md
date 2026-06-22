---
title: "Paper Digest: 2026-06-22"
categories: [Paper Digest]
tags: [AI, Coding Agents, Software Engineering, AGENTS.md, SWE-bench]
---

今天最值得看的 paper，我会选 **Probe-and-Refine Tuning of Repository Guidance for Coding Agents**。

这篇论文处理的问题非常贴近现在的 coding agent 使用现场：一个 repo 里的关键操作知识，往往不在代码里。

比如：

- 哪些文件负责哪些 subsystem
- 测试应该怎么跑
- 哪些 workflow 容易导致错误修复
- repo 里有哪些历史坑
- agent 应该先看哪里，少走哪些弯路

这些东西通常会被写进 `AGENTS.md`、`CLAUDE.md`、Cursor rules、repo README，或者公司内部的 agent playbook。问题是，这类 guidance 到底有没有用，一直很难判断。

论文给出的答案很干净：关键在于 guidance 是怎么产生和迭代的。

作者提出了 **probe-and-refine tuning**。它会先生成 synthetic bug-fix probes，用这些 probe 去诊断当前 repo guidance 哪里缺信息、哪里误导 agent、哪里没能帮助 agent 定位到正确文件。然后系统用 single-shot LLM calls 去修改 guidance 文件。

注意，这个 tuning 阶段不跑完整 agent loop，也不让模型实际用工具修 bug。它更像是在 agent 真正上场前，先用一批诊断题检查说明书有没有写清楚。

实验结果挺直接。

在 SWE-bench Verified 上，使用 Qwen3.5-35B-A3B、200 step budget，probe-and-refine 达到 **33.0% mean resolve rate**。作为对比，初始化用的 static knowledge base 是 **28.3%**，没有 guidance 的 baseline 是 **25.5%**。两个差异都是统计显著的。

更有意思的是，提升主要来自 coverage。

refined guidance 让 agent 多产生了 **14.5 percentage points** 的 evaluable patches，但 per-patch precision 基本保持在约 **59%**。这说明 guidance 的主要作用，是帮助 agent 找到该看的地方、让它能完成更多可评估的尝试。patch 本身的命中率并没有突然变高。

这个结论很实用。

很多人写 repo instructions 时，容易把它当成一份静态文档：列一堆目录结构、命令、注意事项，然后希望 agent 自己理解。但 coding agent 真正需要的，是能帮助它在有限 step budget 内更快到达关键上下文的信息。

换句话说，好的 `AGENTS.md` 应该用任务表现来校准。

如果某类 bug-fix probe 总是让 agent 找错文件，说明 guidance 里缺 subsystem mapping。如果 agent 总是跑错测试，说明 test workflow 没写到可执行的粒度。如果 agent 在更大 step budget 下反而乱走，说明 guidance 没有帮它把额外步骤花在有效搜索上。

这篇 paper 对我来说最有价值的一点，是它把 agent guidance 变成了一个可以优化的工程对象。

`AGENTS.md` 不应该只靠人类经验一次性写完。它可以被 probe，可以被诊断，可以根据 agent 的失败模式持续修。

这也很适合 OpenClaw / Codex 这类工作流。我们已经在大量依赖 workspace instructions、skills、memory、tool notes。下一步真正值得做的，是用真实或合成任务去反向测试这些 instructions：它们到底有没有让 agent 少走弯路，有没有帮助 agent 更快定位文件，有没有减少重复错误。

今天第二篇是 **Phoenix: Safe GitHub Issue Resolution via Multi-Agent LLMs**。

Phoenix 是一个覆盖 GitHub issue triage、修复验证和 PR creation 的多 agent 系统。它把流程拆给 planner、reproducer、coder、tester、failure analyst、PR agent，并用 GitHub webhook state machine 协调。系统里有七层 safety controls，所有改动都要和 baseline test run 做比较，再决定是否开 PR。

这篇值得看的地方，是它不像一个只跑 benchmark 的 demo，更像一个真实部署系统的复盘。论文提到了 WAF filtering、token expiry、permission boundaries、flaky CI、path localization 错误这些实际问题。报告里最诚实的一点是，42 个真实 issue 的 pilot 虽然保持了 100% correctness preservation，但人工检查发现大约只有一半 PR 是 well-targeted fixes。

这对 GitHub-native coding agents 很重要：安全门、baseline tests、状态机和失败分析，都是系统的一部分。

第三篇是 **AutoPass: Evidence-Guided LLM Agents for Compiler Performance Tuning**。

AutoPass 做的是 compiler performance tuning。它没有把 compiler 当黑盒，而是让 LLM agent 查询 compiler-internal optimization states、分析 intermediate representation，再结合 runtime measurement 去迭代优化配置。它是 inference-only、training-free，在 LLVM 上相对 `-O3` 报告了 x86-64 的 1.043x 几何平均加速，以及 ARM64 的 1.117x。

这篇的启发是：agent 要进入复杂系统，不能只靠泛泛的自然语言推理。它需要能看见系统内部证据，然后用这些证据限制搜索空间。

把三篇放在一起，今天的信号很集中：

coding agent 的瓶颈越来越多地出现在 repo-level 和 system-level context 上。模型能力当然重要，但 agent 能不能看见正确证据、使用正确 guidance、通过正确安全门，决定了它能不能在真实工程环境里稳定工作。

今天先读 `2606.20512`。如果你关心 `AGENTS.md`、coding-agent workflow、SWE-bench、repo instructions，或者怎么让 agent 少走弯路，这篇值得拆。

## 今日 3 篇精选

### 1) Probe-and-Refine Tuning of Repository Guidance for Coding Agents
- 链接: https://arxiv.org/abs/2606.20512
- 摘要速读: 用 synthetic bug-fix probes 诊断并迭代 repo guidance，让 `AGENTS.md` 这类说明文件更能帮助 coding agent 定位正确文件和使用 step budget。
- 为什么重要: 它把 repository guidance 变成了可以测、可以调、可以通过任务表现校准的工程对象。

### 2) Phoenix: Safe GitHub Issue Resolution via Multi-Agent LLMs
- 链接: https://arxiv.org/abs/2606.20243
- 摘要速读: 一个覆盖 GitHub issue triage、修复验证和 PR creation 的多 agent 系统，用 webhook state machine、baseline tests 和多层 safety controls 管住自动修复流程。
- 为什么重要: 它展示了 coding agent 真正部署时会遇到的系统问题，包括权限、CI、token、WAF、path localization 和 PR 质量。

### 3) AutoPass: Evidence-Guided LLM Agents for Compiler Performance Tuning
- 链接: https://arxiv.org/abs/2606.20373
- 摘要速读: 让 LLM agents 读取 compiler internals 和 runtime evidence，迭代搜索 LLVM optimization configurations。
- 为什么重要: 它展示了 agent 在复杂技术系统里如何依赖可验证证据来约束搜索空间。

## 一句话结论

今天最强的研究信号是，**coding agent 的真实能力，越来越依赖它能否获得、校准并使用高质量的 repository/system context。** `2606.20512` 值得先读。
