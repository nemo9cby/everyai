---
title: "Paper Digest: Coding Agents — Training, Recovery, and Self-Programming"
date: 2026-02-20 09:00:00 -0500
categories: [Paper Digest]
tags: [ai, coding-agents, llm, software-engineering]
---

6 papers deep-read + 4 quick scans on the latest in coding agents and LLM-powered software engineering.

## 1. Hybrid-Gym: Training Coding Agents to Generalize Across Tasks

📄 [arxiv.org/abs/2602.16819](https://arxiv.org/abs/2602.16819)

**Core problem:** Existing coding agent evaluations (like SWE-Bench) only focus on single GitHub issue fixes, but real-world agents need to handle diverse tasks: code exploration, testing, architecture design, etc.

**Method & Results:** Hybrid-Gym decomposes agent trajectories into fine-grained components, identifies cross-task transferable skills (function localization, dependency search), and designs scalable synthetic training tasks. After training on synthetic tasks, agents significantly improve on real tasks: **SWE-Bench Verified +25.4%**, SWT-Bench Verified +7.9%, Commit-0 Lite +5.1%.

**Why it matters:** This is a rare work that improves real SE task performance through synthetic task training. The +25.4% SWE-Bench improvement is remarkable. Hybrid-Gym complements downstream datasets (e.g., SWE-Play adds another +4.9% on SWT-Bench).

---

## 2. Wink: Recovering from Misbehaviors in Coding Agents

📄 [arxiv.org/abs/2602.17037](https://arxiv.org/abs/2602.17037)

**Core problem:** Production coding agents exhibit misbehaviors in ~30% of trajectories — specification drift, reasoning problems, tool call failures.

**Method & Results:** Wink is a lightweight async self-intervention system that observes agent trajectories and provides targeted corrective guidance. Tested on 10,000+ real trajectories, it successfully resolves **90% of single-intervention misbehaviors**. A/B testing shows significant reduction in tool call failures, per-session token consumption, and human intervention needs.

---

## 3. OpenSage: Self-programming Agent Generation Engine

📄 [arxiv.org/abs/2602.16891](https://arxiv.org/abs/2602.16891)

**Core problem:** Existing Agent Development Kits rely on human-designed agent topologies, tools, and memory components.

**Method & Results:** OpenSage is the first ADK that lets LLMs automatically create agents — supporting self-generated topologies and toolsets. Agents can autonomously create and manage sub-agents and tool packages, with a hierarchical graph-structured memory system. Outperforms existing ADKs on three SOTA benchmarks.

---

## 4. What to Cut? Predicting Unnecessary Methods in Agentic Code Generation

📄 [arxiv.org/abs/2602.17091](https://arxiv.org/abs/2602.17091)

**Core problem:** Agentic coding tools (Copilot, Cursor) shift the review burden from implementers to reviewers. A significant portion of AI-generated code gets deleted during review.

**Method & Results:** A prediction model identifies functions likely to be deleted during PR review, achieving **AUC 87.1%**. First study specifically targeting the "over-generation" problem in AI-generated code.
