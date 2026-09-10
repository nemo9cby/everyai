---
title: "Paper Digest: 2026-09-10"
categories: [Paper Digest]
tags: [AI, LLM, Post-Training, Code Agents, Agent Harness, On-Policy Learning]
---

今天最值得看的 paper，我会选 **Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails**。

论文研究一个很容易被低估的问题：当 agent harness 已经针对某个模型优化过，再用强模型的成功轨迹微调这个弱模型，效果会怎样？

答案相当反直觉。Harness optimization 把 Qwen3-Coder-30B-A3B 在七个 enterprise tasks 上的平均成功率从 29.2% 提高到 78.0%。接着用更强的 Gemini expert 完整轨迹做 LoRA-SFT，平均成功率降到 63.1%，七个任务全部退步，平均损失 14.9 个百分点。

问题出在 model-harness fit。Fine-tuning 确实把 expert knowledge 和 scaffold usage 教给了学生，同时也改变了学生的 planning style。原来的 harness 是围绕学生自身的执行习惯优化出来的。学生开始模仿 expert 的规划方式后，原本有效的 prompt、tool recipe、hook 和 finish cadence 无法继续配合它。

## Harness 和 weights 共同定义 agent

论文使用七个可客观验证的 enterprise tasks，覆盖 payroll auditing、budget approval、stock alerting、IoT anomaly detection、browser automation、website management 和 code refactoring。

Harness optimization 采用 GEPA-style search，由 meta-agent 根据失败案例修改五类组件：

- system prompt
- tools
- execution hooks
- context management
- sub-agent configuration

只有能提高 validation performance 的修改才会保留。经过这一轮适配，Qwen 的 mean test success 从 29.2% 到 78.0%，提升 48.8 个百分点。

有趣的是，这套针对 Qwen 搜出来的 harness 也能帮助更强的 Gemini。Gemini 在 base harness 上的平均成功率是 84.4%，使用 evolved harness 后达到 93.6%。它在 93.6% 到 100% 的 rollouts 中触发了 harness 新增的组件。

这说明 harness 学到的 domain recipe 有真实价值，也留下了 15.6 个百分点的 student-expert gap。最自然的下一步看起来是让 Qwen 模仿 Gemini 的成功轨迹。

## 完整 expert trajectory 为什么会伤害学生

作者收集 Gemini 在 evolved harness 下的成功轨迹，转换成 Qwen 的 chat 和 tool-call format，再混入 Qwen 自己的成功样本做 LoRA-SFT。

结果从 78.0% 降到 63.1%。七个任务全部回退，其中 payroll auditing 下降 29.9 个百分点，website management 下降 20.3，browser automation 下降 16.0。

一个关键 ablation 让原因更清楚。同一套 imitation recipe 放在未优化的 base harness 上，Qwen 从 29.2% 提高到 35.5%。Expert data 本身包含有效能力，损失发生在 fine-tuned model 与 evolved harness 的配合上。

Failure analysis 进一步区分了 knowledge failure 和 planning failure。Imitation 之后，学生更频繁地使用 harness 提供的 domain recipe，知识确实传进去了。与此同时，planning failures 在失败样本中的占比显著上升到 14.6%。它会采用 expert 的规划结构，却没有足够能力稳定执行这套结构，最后出现 premature stopping、错误恢复失败或重复 replanning。

这对 agent post-training 很重要。一个 deployed agent 的行为分布由 weights、prompt、tools、hooks 和 context policy 一起决定。只评估训练 loss 或单独评估模型能力，可能看不到系统层面的兼容性损失。

## On-policy correction：只改失败的那一步

论文提出一个更克制的数据合成方法。流程从 Qwen 在 evolved harness 下的自身 rollouts 开始：

1. 一个 meta-level MLE agent 定位失败 trajectory 中最早出错的 turn。
2. 强 expert 只重写这一轮。
3. 其余上下文、planning steps 和 tool interaction 保持学生原样。
4. 用这些 minimally edited trajectories 做 LoRA-SFT。

这样构造的数据仍然位于学生实际访问的 states 上，也保留了学生原有的 planning distribution。

On-policy correction 把平均成功率从 78.0% 提高到 79.7%。增幅不算巨大，但它在七个任务上都没有造成实质回退，其中五个任务提升。Website management 增加 5.6 个百分点，stock alerting 和 code refactoring 各增加 2.2，anomaly detection 增加 1.7，browser automation 增加 1.2。

更重要的是，planning failure 占比只从 1.1% 变到 1.8%，接近原模型水平。Knowledge failure 从 46.2% 降到 43.2%，overall failure rate 也从 28.9% 降到 26.8%。训练耗时低于一小时，因此可以作为迭代式 agent improvement loop 的一部分。

## 对 code-agent post-training 的启发

这篇论文给出一个很实用的评估单位：不要只看 model checkpoint，也要看 checkpoint 与当前 harness 的组合。

每次修改 system prompt、tool schema、memory policy、sub-agent topology 或 weights 后，都应该重新测量：

- task success 是否变化
- planning failure 与 knowledge failure 如何变化
- 模型是否仍按预期使用 harness components
- tool-call sequence 和 stopping behavior 是否发生漂移
- training data 是否覆盖部署模型真实访问的 states

对于数据构造，论文支持一条具体原则：从当前学生的失败 rollout 出发，定位失效点，再引入局部 expert correction。这样能把 supervision 放在部署分布上，也减少对成功 planning structure 的无关改写。

这项工作仍有明确边界。On-policy correction 只带来 1.7 个百分点的平均增益，没有关闭 Qwen 与 Gemini 的差距。七个任务来自同一 enterprise benchmark suite，harness search 和 correction localization 又依赖强 meta-agent。后续需要验证这种兼容性问题在更开放的 coding environments、不同 harness optimizer，以及 RL 更新下是否同样成立。

## 另外两篇

第二篇是 **Φ-Bench: Can Large Language Models Engineer the Infrastructure That Powers Them?**。

它面向 LLM infrastructure engineering 设计 benchmark，覆盖从局部 kernel completion 到长链路实现和 end-to-end system optimization 的任务。它更接近真实 infrastructure 工作：agent 需要同时维护 correctness、定位 bottleneck，并在 repository context 中完成可验证的性能改进。

第三篇是 **SWE-Bench Pro Verified: A Reliable Benchmark for Software Engineering Agents**。

作者审计 SWE-Bench Pro，处理两类会夸大 agent 能力的问题：gold solution 或 hidden evaluation information 泄漏带来的 reward hacking，以及描述误导、test scope 错误等 task-quality defects。重新验证后，一些模型的成绩明显下降。

论文：<https://arxiv.org/abs/2609.09134>

另外两篇：<https://arxiv.org/abs/2609.10226>、<https://arxiv.org/abs/2609.08149>
