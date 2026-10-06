---
title: "Paper Digest: 2026-10-06"
categories: [Paper Digest]
tags: [AI, LLM, Coding Agents, SWE-bench, Agent Skills, Test-time Training]
---

今天最值得看的 paper 是 **Correct Code, Broken Contributions? SWE-CC: Benchmarking Repository Policy Compliance for Coding Agents**。

现在的 coding-agent benchmark 通常问一个问题：patch 能不能通过测试？

SWE-CC 加入了另一个现实问题：这份贡献符合 repository 自己的开发规则吗？

一个 agent 可以正确修复 issue，同时违反格式规范、git 流程、测试要求、生成文件约定或其他项目政策。测试给出绿色结果，maintainer 仍然无法放心 merge。

## 把 repository policy 变成可执行检查

作者从 12 个成熟开源项目的贡献文档中提取规则，整理成 823 条 machine-checkable atomic policies，再把这些检查接到 500 个 SWE-bench Verified end-to-end tasks 上。

每条 policy 都尽量缩小成一个确定性 checker。这样，evaluation 可以同时观察两类结果：

- patch 是否完成原始功能任务
- agent 是否遵守 repository-specific contribution policy

SWE-CC 还记录 agent runtime behavior。检查范围覆盖完整 trajectory 和最终 deliverable，能够看到 agent 在中间步骤运行了什么命令、修改了什么文件、是否跳过规定流程。

这个设计很关键。许多违规会在后续步骤中被覆盖，最终 diff 看起来干净，执行过程却已经越过了项目边界。

## 功能正确之后，仍有 43.1% policy 被违反

论文评估了四个 LLM 和两种 agent scaffold。结果显示，即使 agent 生成了 functionally correct patch，仍然违反 43.1% 的 applicable policies。

更值得注意的是，接近一半的违规发生在 intermediate execution steps。只检查最终代码，会漏掉 agent 如何得到这份代码，以及过程中对 repository 做过什么。

这让 coding-agent evaluation 多出一条独立坐标轴：

1. Task competence，能不能解决 issue
2. Contribution compliance，能不能按项目接受的方式解决 issue

两者不能互相替代。测试通过证明行为结果的一部分，repository governance 约束工程过程、可维护性和团队协作成本。

## AGENTS.md 需要对应 verifier

今天很多 agent 会读取 `AGENTS.md`、`CONTRIBUTING.md`、CI configuration 和 tool policy。把规则写进 prompt 只能完成第一步。生产系统还需要知道规则有没有真正执行。

一个更完整的 pipeline 可以这样设计：

1. 从项目文档和配置中抽取 atomic policies
2. 能确定性检查的规则直接编译成 executable checks
3. 在相关步骤加载规则，避免把 823 条约束塞进每次 prompt
4. 同时记录 tool trajectory、workspace state 和 final diff
5. 分开报告 functional success 与 compliance
6. 把违规定位到 planning、tool use、intermediate mutation 或 final packaging

这里最难的工程问题是 selective enforcement。大型 repository 的规则很多，某个任务实际只涉及其中一小部分。系统需要根据文件、命令和 task scope 动态激活 policy，同时保证漏召回不会变成绕过规则的通道。

## 对 code-agent post-training 的意义

SWE-CC 也提供了一类新的训练信号。

现有 code-agent SFT 和 RL 数据通常围绕 test outcome 筛选 trajectory。加入 policy checker 后，一条轨迹可以获得更细的标签：功能完成、流程合规、最终状态合规，以及具体违反了哪条 repository rule。

这些信号可以用于：

- 过滤“测试通过但过程危险”的 SFT trajectories
- 为 RL verifier 增加 repository-local constraints
- 训练 agent 在执行前识别适用规则
- 对中间 mutation 做 credit assignment
- 区分模型错误、harness 错误和 policy retrieval 错误

同时也要防止 agent 学会迎合 checker。Policy suite 应包含语义等价、轨迹审计和 held-out rules，避免系统只记住一组表面模式。

SWE-CC 最有价值的提醒很朴素：**能修好 bug，只代表 coding agent 通过了工程工作的第一关。**

## 今天另外两篇相关论文

**SkillScriptBench** 评估 agent 能否维护同时包含 Markdown instruction 和 executable scripts 的 skill packages。它从超过 35,000 个 GitHub skill roots 中选出 100 个 package，构建 350 个 repair 与 preservation tasks。AST-Guided Skill Revision 通过语法树和调用关系限制修改范围，把 repair success 提高 21.9 到 27.7 个百分点。对于 OpenClaw、Claude Code 和 Codex 这类 skill ecosystem，这几乎是一份现成的安全自进化测试规格。

**ASCENT** 研究 long-horizon agent 的 online test-time training。直接学习单次 self-generated trajectory 容易让 policy 失稳。ASCENT 让 frozen initial model 读取 verified trajectory 作为 privileged hindsight，再把它的 token distributions 蒸馏进 persistent LoRA fast weights。Agent 因此可以把跨任务经验写进参数，同时保留一个稳定 teacher 提供 dense supervision。

论文：<https://arxiv.org/abs/2610.06193>

另外两篇：<https://arxiv.org/abs/2610.04008>、<https://arxiv.org/abs/2610.05303>
