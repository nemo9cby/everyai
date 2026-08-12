---
title: "Paper Digest: 2026-08-12"
categories: [Paper Digest]
tags: [AI, Code Agents, Software Engineering, Multi-Agent Systems, Long-Horizon Agents, Agent Infrastructure]
---

今天最值得看的 paper，我会选 **Persistent Recursive Worlds Enable Autonomous Software Evolution**。

它研究的是 coding agents 最难绕开的系统问题：一个软件项目可能持续几天、几个月，甚至几十年，任何单个 agent 的 session、context 和 memory 都活不了这么久。作者提出的 EvoX Genesis 把连续性放进 software project 本身。代码、测试、约束、path-specific context 和 Git history 持续存在，执行具体任务的 agents 可以不断结束和重新实例化。

论文最醒目的实验是一场持续 123.4 小时的 compiler development run。DeepSeek V4 Flash 通过 1,019 个 archived agent episodes，从几乎空白的 repository 开始，构建了一个约 24.9 万行的 Rust C compiler。它通过了 220/220 个 c-testsuite cases、32/36 个 LLVM cases、93/93 个执行过的 Csmith programs，以及 2,904 个 Rust tests。记录的 model-token charge 是 44.38 美元。

这个数字很抓眼球，真正值得研究的部分是背后的组织方式。

## Project 才是连续性的载体

Genesis 用两个坐标定义 local software world：一个 accepted version 和一个 repository-relative path。

Accepted version 决定当前可继承的完整项目状态与历史，path 则限定 agent 的局部责任范围。一个有限寿命的 agent 进入这个 local world，执行 bounded task，在独立 branch 或 worktree 中提出修改。它还可以把子任务递归委派给另一个 path 下的 child agent。

Delegation 本身不会改变项目。只有经过测试、约束检查和 parent review 后被接受的 change，才会形成新的 software event，推进 accepted version history。Rejected change 消失，失败信息仍可以在后续 episode 中被整理成有用的上下文。

这个设计把 reasoning 和 persistence 分开了。Reasoning 留在短命 agents 中，persistence 留在 repository、tests、accepted commits 和 controlled context records 中。下一批 agents 无需继承上一批 agents 的完整 conversation，只需要从一个可执行、可验证、带历史的 project state 继续工作。

## 1,019 个 episodes 如何累积成 compiler

Compiler experiment 从只含 `.gitignore` 和 `genesis.toml` 的 repository 开始。目标包括 Clang-compatible CLI、标准 object-file 和 linker integration、LLVM IR export、C11 支持，以及 x86 和 x86-64 back ends。系统不能直接使用或翻译 Clang/LLVM source。

Managers 把 root objective 分解到不同 repository paths，local agents 分别处理 front end、type checking、IR、optimization、back end、integration 和 validation。Parent agents 审查返回的修改，accepted commits 随后成为其他 agents 的起点。

最终 repository 有 248,989 行、750 个 tracked text files，递归 delegation depth 达到 5。没有任何单个 episode 横跨完整开发过程。早期对 front end 和 IR 的决策通过 accepted code 固化下来，后续 agents 在真实集成环境里暴露问题，再继续修复。

这更像大型开源项目的协作模型：贡献者会离开，maintainers 会更换，持续存在的是 repository、tests、review rules 和 accepted history。

## Agent 和 foundation model 都可以被替换

第二组实验测试 continuity。作者从一个由 GLM 5.2 完成的 compiler repository 出发，建立两条 continuation branches。一条继续使用 GLM 5.2，另一条改用 DeepSeek V4 Flash。

两条分支都能从同一个 completed compiler 继续开发。GLM branch 使用 98 个 agents，DeepSeek branch 使用 178 个 agents，它们形成不同的 delegation depth、commit history、code churn 和 repository size。论文没有把两者包装成 head-to-head model ranking，因为 test sets 和预算并不完全一致。

这个实验说明 accepted project 可以提供共同的过去与共同的 obligations，同时允许不同模型选择不同的未来路径。系统无需把某个特定 manager agent 的隐藏状态当作唯一连续性来源。

## 从 Fortran MESA 到 Rust

第三组实验把 Genesis 用在 existing scientific software redevelopment 上。系统读取 MESA 的 13 个 module directories，覆盖 139,414 行 Fortran 代码与 module-level tests，并用 DeepSeek V4 Flash 重写为 Rust workspace。

这场 run 持续 33.22 小时，spawn 了 272 个 agents，生成 89,946 行 Rust，包括 tests 和 benchmarks。Workspace 通过 1,052 个 tests，18 个 tests 被标记为 ignored。六个 audited numerical workloads 中，EOS lookup 与 Newton solve 达到 bit-exact，其余四个 workload 的 relative checksum differences 介于 5.1×10^-15 和 3.1×10^-9。报告环境下，Rust median runtime speedup 为 1.55× 到 6.87×。

这里的验收标准很关键。大量代码生成很容易，保留已有 scientific behavior 困难得多。Numerical checks 把 agent 的目标锚定在软件真正有意义的外部行为上。

## 应该怎样解读这些结果

论文给出的是 capability demonstration，因果证据仍然有限。

每个主要 setting 只有一场记录 run。研究还没有完成 flat organization、recursive delegation、persistent manager 和不同 saved records 之间的 controlled ablation。Archive 也没有完整记录每一次 human action，所以无法把缺少 intervention log 等同于零人工介入。

44.38 美元只包含 foundation-model token charges。Local hardware、storage、controller overhead、networking 和 human labor 都不在其中。24.9 万行也只是 physical line count，包含 comments 和 blank lines，不能直接代表 complexity 或 feature completeness。

这些 caveats 没有削弱系统设计的价值。它们给出了下一轮实验最清楚的方向：固定 code，替换 non-code context；固定 project state，比较 fresh agents 和 persistent agents；固定预算，比较 recursive 与 flat organization；再测量哪些 records 真正提高未来任务成功率。

## 对 code-agent post-training 的启发

Genesis 的 accepted history 天然提供多层训练信号：

- accepted commit 提供 outcome-level supervision
- rejected proposal 和 validation failure 暴露 failure modes
- parent review 记录局部修改为何可以进入全局 project
- delegation tree 描述 task decomposition 与 credit assignment
- later repair 说明早期 commit 在更长 horizon 中产生的后果

训练 pipeline 可以把 repository state 当作每个 rollout 的环境，把 tests 与 integration checks 当作 verifiable rewards。一个难点是 delayed credit：某个 locally correct change 可能几百个 episodes 后才暴露 architectural debt。另一个难点是避免奖励 code volume，最终信号应围绕 externally meaningful behavior、maintainability 和 accepted consequences 设计。

## 为什么今天选它

Genesis 给 long-horizon coding agents 提供了一个很硬的系统抽象：让 project 活得足够久，让 agents 保持有限寿命。

它没有依赖无限 context，也没有要求某个 manager 永久在线。Version history 负责继承，path 负责局部责任，validation 负责筛选能够进入未来的 consequences。这个结构已经支撑超过一千个 agent episodes 累积成一个 interdependent software system，也能跨 foundation-model replacement 继续工作。

如果未来的 coding agents 真能维护多年运行的软件项目，它们大概率需要类似的 project-centered continuity。

## 另外两篇

第二篇是 **One Recipe, Many Harnesses: What Self-Evolution Encodes Across Languages and Models**。

作者把同一套 harness self-evolution recipe 应用到八种 programming languages 和三个 base models。结果显示，evolved harness 主要补偿 recoverable execution defects。不同语言共享抽象 playbook，具体 ecosystem machinery 几乎互不重叠。Shared core 可以 distill 成 universal harness，package manager、compiler 和 test convention 等 ecosystem margin 仍需要 native re-evolution。

第三篇是 **Why Does CLAUDE.md Keep Growing? Catastrophic Remembering in Agentic Coding**。

论文分析 1,867 个 repositories 中 247,694 条 instruction lifetimes，发现 agentic prompts 在生命周期中平均增长 226%，每个 commit 增加 4.9 条 net instructions。旧规则的 rationale 消失后，删除会带来 regression risk，于是 instructions 越积越多。作者发现，为规则保留 rationale comments 可以在 verifiable worlds 中把 excess instructions 从 211.3% 降到 1.4%。这对 CLAUDE.md、AGENTS.md、skills 和长期 agent memory 都很实用。

论文：<https://arxiv.org/abs/2608.10450>

另外两篇：<https://arxiv.org/abs/2608.10178>、<https://arxiv.org/abs/2608.11095>
