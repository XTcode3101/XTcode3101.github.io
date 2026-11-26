---
title: "Most Popular Multi-Agent Architectures Today: Subagent, Agent Teams, and A2A Protocol"
description: "An in-depth comparison of the 3 most popular Multi-Agent architectures: Subagent, Agent Teams, and A2A Protocol — plus a decision matrix and token cost optimization strategies."
date: 2026-11-26 17:40:00 +0700
categories: [AI & Agent, Multi-Agent]
tags: [multi-agent, subagent, agent-teams, a2a, claude-code, architecture]
pin: false
toc: true
media_subpath: /assets/img/posts/
---

Hello tech enthusiasts! While 2023–2024 was the era of Single AI Agents centered around tool calling (Tool Calling / MCP), from 2025–2026 onward, the spotlight of AI Engineering has shifted decisively toward **Multi-Agent Systems** — with practical patterns shaped by **Anthropic** (through Claude Code Subagents & Agent Teams) and **Google & the Linux Foundation** (through the open Agent2Agent - A2A protocol). Understanding how agents coordinate, divide work, and communicate will help you design AI systems that are both powerful and immune to "suffocating" under context window limits.

---

## 1. Overview & Core Differences Among the 3 Multi-Agent Mechanisms

When a problem grows too large for a single AI Agent, our natural instinct is to split the work across multiple agents. But how should you structure that division so agents don't step on each other's toes?

Today, **Multi-Agent Orchestration** revolves around 3 dominant mechanisms: **Subagent**, **Agent Teams**, and **A2A (Agent2Agent Protocol)**. The core difference defining each model lies in its **communication topology** and how work is partitioned:

- **Subagent (Hub-and-Spoke / Parent-Child Model)**: A strict hierarchical structure. Subagents only communicate with their parent (**Parent / Lead Agent**). All task assignments and result reports must pass through the parent; subagents are completely unaware of each other and never communicate directly.
- **Agent Teams (Team Lead - Teammates with Shared Coordination Memory)**: Keeps the **Team Lead** and **Teammates / Workers** relationship, but removes communication barriers. Every member — both Lead and Workers — can directly read and update a **Shared Task List** to claim or update work, and use a **Mailbox** to message each other directly.
- **A2A (Agent2Agent Protocol — Peer-to-Peer Model)**: Breaks out of fixed parent-child sessions toward peer-to-peer collaboration between a **Client Agent** and a **Remote Agent**. An agent can dispatch individual `Task`s over the network to another independent agent and receive deliverables (`Artifact` / `Message`) back.

With this high-level map of the three architectures in mind, let's pop the hood on each mechanism to see how they work internally and what trade-offs they involve.

---

## 2. Mechanism 1: Subagent — Fast, Lightweight, and Context-Isolated Like a Tool

When moving from a Single Agent to Multi-Agent, **Subagent** is almost always the first stop thanks to its simplicity and effectiveness. The diagram below illustrates the one-way hierarchy between a Parent Agent and its child Subagents:

![Subagent Model (Hub-and-Spoke)](pasted-image-20261126111146.png)
*Subagent Model (Hub-and-Spoke): The Parent Agent retains full control over spawning, delegating, and collecting results from child agents.*

### How It Works
Imagine a Parent Agent tackling a large problem that requires reading 40 log files to pinpoint a bug. Reading them all itself would immediately flood its context window with noise. Instead:

1. The Parent Agent spawns child agents with an **isolated context window** to execute a specific task, receiving only a concise summary back. If you build this yourself from scratch, the simplest practical approach is spawning a CLI process with a prompt via `child_process` or wrapping it as an MCP tool.
2. From the Parent Agent's perspective, invoking a subagent looks just like calling any regular execution tool (such as a `Bash tool` or `Agent tool`): pass inputs, wait for completion, and read the output.

### Benefits
- **Context Isolation / Preservation**: All raw data and trial-and-error stay confined inside the child's context, keeping the parent's context clean and lean.
- **Parallel Execution**: The Parent Agent can launch multiple subagents simultaneously to handle independent tasks.
- **Dead-Simple Implementation**: Wrap it as a tool or an MCP tool and your system is up and running immediately.

### Trade-offs to Watch
While lightweight, on long-horizon problems the one-way parent-child structure of Subagents starts showing cracks:
- **Higher token cost and latency for cross-agent coordination**: For complex tasks (e.g., a UI Agent needing API specs from a Backend Agent), they cannot ask each other directly. Every message must route through the Parent Agent as a middleman, rapidly inflating token usage.
- **Knowledge bottleneck at the Parent Agent**: Because the task list and plan live solely with the Parent Agent, it must shoulder the entire project's coordination context alone.
- **Lifecycle management burden**: The Parent Agent must directly manage the full **lifecycle** of every subagent from spawn to teardown.

---

## 3. Mechanism 2: Agent Teams — When Agents Read a Shared Board and Message Each Other

If routing everything through the "boss" (Parent Agent) creates a communication bottleneck in Subagents, a natural question follows: *Why not give agents a shared task board and a private chat channel so they can coordinate directly?* That is exactly why **Agent Teams** was born. In the architecture diagram below, notice how collaboration changes with the introduction of a Shared Task List and Mailbox:

![Agent Teams Model with Task List and Mailbox](pasted-image-20261126111019.png)
*Agent Teams Model: Both Team Lead and Workers interact directly with the shared Task List and communicate peer-to-peer through the Mailbox system.*

### How It Works
In Agent Teams, the workspace is organized just like a real-world software engineering team:

1. The **Team Lead** spawns **Teammates** as independent sessions (e.g., separate CLI processes) alongside a shared **Task List** directory. Inside this directory, multiple `task.json` files track task details and which teammate owns them (for example, stored at `~/.claude/tasks/{team-name}/` in Claude Code).
2. To communicate, each agent gets its own inbox file (`mailbox.json`, e.g., `~/.claude/teams/{team-name}/inboxes/{agent-name}.json`). When one agent needs to talk to a teammate, it writes a message straight to the recipient's mailbox without bothering the Team Lead.

### Benefits
- **Maximum context isolation**: Each teammate runs in an independent session with a completely separate context window.
- **Effective parallel execution**: Large workstreams can proceed concurrently without blocking each other.
- **Flexible discussion and plan updates**: The biggest advantage of Agent Teams is that agents can discuss requirements peer-to-peer, **self-claim** tasks, or propose adding/updating tasks on the Task List as reality evolves.

### Trade-offs to Watch
Granting autonomy and free-form communication to agents significantly increases engineering complexity:
- **Risk of "over-meeting" without tight prompting**: Requires carefully crafted prompts specifying when and what to communicate. Without guardrails, agents will chat back and forth endlessly even on trivial tasks, burning tokens and hurting efficiency.
- **Concurrency & Race Conditions**: If building this yourself, you must implement solid concurrency controls — such as handling two or more agents attempting to edit the same Task List file simultaneously.
- **Lifecycle management**: Even though workers are more autonomous, the Team Lead remains responsible for monitoring and managing the lifecycle (spawning, tracking, shutting down) of all teammates.

---

## 4. Mechanism 3: A2A (Agent2Agent Protocol) — Peer-to-Peer Networking Across Agent Ecosystems

Subagent and Agent Teams solve coordination inside a single machine or toolset very well. But what happens when your Agent needs help from a specialist Agent living on another server, written in a completely different language or framework? That's where **A2A (Agent2Agent Protocol)** steps in. The illustration below shows how independent agents discover each other via Agent Cards and dispatch tasks directly:

![A2A Protocol Peer-to-Peer Model](pasted-image-20261126111112.png)
*A2A Protocol Model: Agents discover each other's capabilities via Agent Cards and communicate peer-to-peer to delegate individual Tasks.*

### How It Works
Instead of spawning child processes from a single parent, A2A treats each agent as an independent network entity:

1. Each agent publishes a capability profile called an **Agent Card** (`agent-card.json`) describing its capabilities, `skills`, input/output formats, and endpoint URL. You can register Agent Cards on a central A2A Registry/Catalog, or let each agent act as an independent A2A Server publishing its card at the standard `/.well-known/agent-card.json` path.
2. When an agent needs help, it inspects **Agent Cards** to pick the right specialist, then sends a message with a `Task` directly to that agent's endpoint.

### Benefits
- **Deep specialization delegation**: Easily delegate work to specialist agents regardless of their tech stack or hosting infrastructure.
- **Wide-scale parallel coordination**: A Client Agent can dispatch parallel tasks across dozens of different Remote Agents.
- **High scalability and reusability**: Once a specialized agent is built and publishes its Agent Card, countless other agents and systems can reuse it.

### Trade-offs to Watch
Designed as a cross-network protocol between independent entities, A2A has natural limitations compared to Agent Teams:
- **No built-in Shared Taskboard / Orchestration Backlog**: In A2A, every Remote Agent is an **opaque agent** unaware of your project's big picture; it simply receives a `Task`, processes it, and returns the result. While A2A strictly manages the lifecycle of each individual `Task` (`SUBMITTED`, `WORKING`, `INPUT_REQUIRED`, `COMPLETED` via `GetTask`, `ListTasks`, `CancelTask`), it does not provide a shared `tasklist` where agents self-organize.
- **More custom orchestration for large projects**: Using A2A for large multi-step projects with complex dependencies requires the caller (Client/Orchestrator) to build a **custom orchestration layer** to plan and stitch workflows together.

---

## 5. Practical Playbook: Choosing the Right Model, Cost Control & Configuration

Having walked through all 3 mechanisms, it's clear there is no single "silver bullet" architecture for every problem. To help you apply them in production without blowing through your token budget, here are 3 key takeaways on selection, cost optimization, and setup.

### Decision Matrix: Which Mechanism Fits Which Problem?
- **Use Subagent when**: Subtasks are independent, you only care about the final result (e.g., scanning 30 source files for bugs, running unit tests, summarizing docs), and you want minimal complexity and cost.
- **Use Agent Teams when**: Multiple agents collaborate on a large repo or project split across layers (Frontend, Backend, Tests) or debate competing debugging hypotheses, where one worker's findings directly reshape another worker's plan.
- **Use A2A when**: You need to connect agents **across different ecosystems, teams, or servers** (e.g., a customer-support Agent on Cloud A calling an inventory Agent built with LangGraph on Server B).

### Token Cost & `Model Tiering` Strategy
A practical reality that often catches newcomers to Multi-Agent systems off guard is token burn rate:
- **Cost Multiplier**: According to real-world data from Anthropic's Multi-Agent systems, multi-agent architectures can consume roughly **~15x** more tokens than a standard single-turn chat (and ~3–4x more than a Single Agent).
- **Optimize with `Model Tiering`**: Never assign your most expensive model to every agent in the system. The smart strategy is using your strongest reasoning model (e.g., `Opus`) as the **Team Lead / Orchestrator** for planning and synthesis, while assigning faster, more cost-effective models (e.g., `Sonnet` or `Haiku`) to the **Subagents / Workers** running in parallel underneath.

### Standard Setup for Agent Teams / Subagents in Claude Code
If you want to experience the difference between Subagent and Agent Teams firsthand in Claude Code, here are the two core configurations:
- `.claude/agents/<agent-name>.md`: Defines a reusable Subagent used both in standard Subagent mode and as a Teammate in a team (declaring `name`, `description`, `tools`, and `model` in YAML frontmatter).
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`: The environment variable (set in your shell or `settings.json`) that enables Agent Teams in interactive terminal sessions.

---

## Conclusion

Looking at the bigger picture, the evolution from **Subagent** → **Agent Teams** → **A2A Protocol** mirrors how humans organize work: starting with an individual delegating to personal assistants (Subagent), scaling up to a project squad sharing a Kanban board and messaging each other (Agent Teams), and finally interconnecting independent departments and companies through a standardized protocol (A2A).

Each mechanism has its own sweet spot:
- Need something lightweight, fast, and easy to isolate context: choose **Subagent**.
- Need to tackle large projects where teammates discuss and re-balance tasks together: choose **Agent Teams**.
- Need scale and cross-server, cross-platform agent interoperability: choose **A2A**.

Where is your current project at — Single Agent, custom MCP/CLI Subagents, or already experimenting with Agent Teams and A2A? Share your experience and real-world use cases below! Wishing you smooth and effective AI Agent fleets!
