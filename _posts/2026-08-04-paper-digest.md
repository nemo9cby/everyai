---
title: "Paper Digest: 2026-08-04"
categories: [Paper Digest]
tags: [AI, Agents, Code Agents, Agent Harness, Long-Horizon Tasks, Evaluation]
---

今天最值得看的 paper，我会选 **LongHorizon-Harness: Advancing Long-Horizon Agents for Real-World Tasks**。

它抓住了 long-horizon agent 的一个核心故障源：任务越长，execution history 越像一份混杂了事实、猜测、计划和自我评价的数据库。模型需要从不断增长的 context 里判断哪些要求已经满足、哪些动作真实发生、哪些结论只是上一轮 agent 的误判。一次错误的 self-assessment 被写进后续推理后，局部失误会逐轮放大。

LongHorizon-Harness 给出的方案叫 Manage-Execute-Audit（MEA）。它把 planning、environment mutation 和 verification 拆成三个职责，并规定只有独立审计过的事实可以更新持久状态。

## 三个角色，一条状态边界

每一轮从 manager 开始。

Manager 读取原始任务、当前 task state 和历史 audit reports，选择一个尚未解决的 subtask。它同时写清 dependencies、constraints 和 acceptance criteria。Manager 没有修改环境的工具权限，因此不能一边规划一边偷偷把自己的推测当成已完成事实。

Executor 接收这个 bounded contract，在 fresh context 中执行。它是唯一可以主动修改环境的角色。Fresh context 很重要，它让 executor 专注于当前 subtask，避免原始长轨迹中的噪声和错误承诺持续干扰行动。

Auditor 随后使用 read-only tools 检查真实环境。它看得到任务、当前状态、subtask contract 和相关审计记录，但看不到 executor 的 raw trajectory。这个隔离减少了 confirmation bias。Executor 说“完成了”只是一条 claim，auditor 在文件、终端、应用状态或测试结果中看到的证据才有资格进入下一轮 task state。

整个循环可以写成：

1. Manager 从 verified state 选择一个 bounded subtask
2. Executor 在 fresh context 中改变环境
3. Auditor 独立检查结果
4. Manager 根据 audit report 更新 state，继续、重试、改路或请求用户输入

跨轮次长期保存的是结构化 task state 和 audit reports。Raw execution transcript 不承担长期记忆职责。

## 为什么这个设计有效

普通 agent harness 常把三个问题混在同一条上下文里：下一步做什么、刚才做了什么、任务是否完成。模型既当运动员又当裁判，还要在越来越长的聊天记录里维护记分牌。

MEA 给系统增加了一条 evidence boundary。Task state 中的 requirement、artifact 和 fact records 必须由环境观察支撑。一次执行失败可以被审计发现并触发 recovery；一次局部成功也能被稳定保留，不必依赖下轮模型从旧 transcript 中重新理解。

这种结构尤其适合跨 GUI 与 CLI 的工作流。Agent 可能在浏览器中配置服务，再回到 terminal 检查文件和进程，然后进入桌面应用验证结果。环境状态分散在多个界面中，仅靠语言模型的 working memory 很容易失真。独立 auditor 把各界面的真实状态重新汇总为可用证据。

## 结果很强，代价也写得很清楚

在相同 Qwen 3.7-Plus backbone 下，LongHorizon-Harness 报告：

- WeaveBench PassRate：51.8% → 80.7%
- Terminal-Bench 2.1 success：69.7% → 77.2%
- OSWorld 2.0 binary completion：2.8% → 8.3%
- OSWorld 2.0 partial score：21.5% → 35.2%

在 34 个 OSWorld 2.0 tasks 的子集上，Claude Opus 4.7 也从约 20% 提升到约 35%。Terminal-Bench 完全在 CLI 中运行，因此收益无法归因于 GUI routing。显著提升来自显式状态、bounded execution 和 independent audit 这套组合。

论文同时给出了 token economics。

Manager 只占总 token 的 2.0% 到 8.1%，显式维护状态本身很便宜。Auditor 占 19.4% 到 38.1%，verification 才是主要新增成本。总消耗会随任务和模型变化：WeaveBench 是 baseline 的 2.3 倍，OSWorld output tokens 是 3.6 倍；Terminal-Bench 反而少用 24%，因为更清晰的子任务与恢复机制减少了无效探索。

OSWorld 上，Qwen 3.7-Plus 的平均 output tokens 从 28.9K 增加到 104K。三倍 completion rate 很吸引人，绝对成功率仍只有 8.3%。这提醒我们，harness 能提高 execution reliability，无法补齐模型缺失的 perception、reasoning 或 coding primitive。

## Harness 也决定 agent capability

论文里最值得记住的判断是：agent capability 属于完整的 model-harness system。

模型决定每一轮能提出什么动作和解法。Harness 决定这些局部能力能否被正确拆分、验证、恢复并累积成最终结果。一个较弱模型配合更可靠的 harness，有时可以超过强模型运行在容易丢失状态的基础框架上。

这个结论对 code agent 很实际。Repo 级任务中的失败经常发生在这些位置：忘记原始约束、相信了未经测试的修改、修复一处后破坏另一处、看到 partial progress 就提前结束。更大的 backbone 会减少一部分错误，verified state machine 直接处理的是另一类 failure mode。

它也提供了更清晰的诊断方法。若 auditor 能发现错误，但 executor 多次无法修复，瓶颈可能属于 model capability。若模型有能力完成局部步骤，系统仍因状态漂移和错误 completion judgment 失败，harness 才是主要修复对象。

## 对 personal agent 和 coding agent 的启发

第一，durable memory 应该保存经过验证的 state delta，而非压缩后的整段叙事。Summary 仍然可能把推测包装成事实，audit report 则有明确的 evidence provenance。

第二，执行者不应该单独决定任务完成。最小可行实现可以很简单：每个 subtask 都带 acceptance criteria，执行后启动一个 fresh-context verifier，只允许 read-only commands，并把验证结果写入结构化状态。

第三，tool permission 本身是架构的一部分。Manager 无权改环境，auditor 无权修复，executor 无权把自己的 report 直接写成 verified fact。权限边界让角色分工真正成立。

第四，verification budget 要动态分配。短小、可逆、容易检查的步骤可以轻审计；跨系统、高风险、会影响后续多个步骤的 mutation 值得投入更多 auditor tokens。论文使用统一角色设计，production harness 还可以进一步按风险调度。

## 为什么今天选它

LongHorizon-Harness 的价值在于，它把“agent 要记得自己做过什么”改写成一个更精确的 systems problem：系统必须维护可验证、可恢复、可追踪的 task state。

长任务的可靠性来自持续积累真实进度。Manager 负责选择下一步，executor 负责改变世界，auditor 负责确认世界真的改变了。三者之间那条证据边界，比继续扩充一段无限增长的 context 更值得投入。

## 另外两篇

第二篇是 **Progressive Agent Skill Generation via Reinforcement Learning**。

Skill-alpha 把 skill generation 做成 sequential editing，并用 rollback reward 比较修改前后 skill 在 anchored query 上的真实 downstream behavior。它绕开了“这段 skill 文本看起来是否合理”的表面指标，直接测量一次小编辑是否提高 agent success。使用 GPT-4o worker 时，它在 CL-Bench 和 tau2-bench 上分别超过最强 baseline 3.3 和 6.7 points。对于从 documents 或 traces 自动沉淀 skills 的系统，这是一条很自然的版本化学习路线。

第三篇是 **SWE-Touch: Benchmarking Coding Agents When Users Touch the Code**。

SWE-Touch 测试 coding agent 工作期间用户修改同一份代码的场景。它把 task-relevant Counter-Edits 注入正在运行的修复轨迹，九个 coding models 在 SWE-bench Verified 上平均下降 7.7 percentage points。常见失败包括没发现 workspace 已变化、留下相互冲突的逻辑，以及直接覆盖用户编辑后没有重新检查和跑 targeted tests。真实 IDE collaboration 天生具有 concurrent state，这个 benchmark 补上了 autonomous SWE evaluation 长期忽略的一块。

论文：<https://arxiv.org/abs/2608.01964>

另外两篇：<https://arxiv.org/abs/2608.01678>、<https://arxiv.org/abs/2608.02499>
