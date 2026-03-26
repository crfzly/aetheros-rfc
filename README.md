# AetherOS — Design RFCs for Multi-Agent Orchestration

> Building a nervous system for AI agent clusters on local infrastructure.

## What is AetherOS?

AetherOS is a **production-tested orchestration framework** for coordinating multiple AI agents across heterogeneous local machines. Unlike cloud-first agent frameworks that treat agents as stateless API calls, AetherOS treats agents as **persistent, stateful nodes** in a distributed system — with their own roles, permissions, memory, and failure modes.

It has been running in production since early 2026, coordinating 15+ agents across two physical machines, handling real business operations daily.

## Why These RFCs?

The current landscape of multi-agent frameworks (CrewAI, AutoGen, LangGraph, etc.) solves the "how do I chain LLM calls" problem. But they largely ignore the hard problems that emerge when agents run **continuously in production**:

- How do agents communicate when they can't share memory?
- How do you recover when an agent (or its host machine) goes down?
- How do you let humans and agents coexist in the same task system?
- How do you prevent an autonomous agent from doing irreversible damage?

These RFCs document our answers — battle-tested through months of real-world operation.

## RFC Index

| RFC | Title | Status | Core Question |
|-----|-------|--------|---------------|
| [001](001-jnp-protocol.md) | JNP: Asynchronous Inter-Agent Communication Protocol | Draft | Why should agents communicate like email, not function calls? |
| [002](002-dual-brain.md) | Dual-Brain Model: Heterogeneous Multi-Machine Architecture | Planned | Why does an agent system need disaster recovery thinking? |
| [003](003-handoff.md) | File-Based Handoff: The Filesystem as Message Bus | Planned | Why is the filesystem the most reliable agent communication bus? |
| [004](004-context-bridge.md) | Context Bridge: Cross-Session Memory Persistence | Planned | How should agent "experience" accumulate and transfer? |
| [005](005-workflow-tiers.md) | Three-Tier Workflow: Risk-Graded Human-AI Collaboration | Planned | How do you design brakes for autonomous agents? |
| [006](006-task-scheduler.md) | Unified Human-Agent Task Scheduling | Planned | Why must future task systems treat humans and AI as peers? |

## Design Principles

1. **Files over APIs** — The filesystem is the most battle-tested, debuggable, and recoverable message bus ever built.
2. **Async over sync** — Agents have different speeds, availability, and failure modes. Synchronous orchestration is a single point of failure.
3. **Orthogonal dimensions** — Signal type (what to do) and priority (when to do it) are independent axes, not conflated categories.
4. **Graceful degradation** — Every node can operate independently. Coordination improves throughput; its absence doesn't cause failure.
5. **Human-in-the-loop by default** — Autonomy is earned through risk classification, not granted by default.

## Architecture at a Glance

```
                    ┌─────────────────────┐
                    │   Human (L3 Admin)  │
                    └──────────┬──────────┘
                               │ Approval / Override
                 ┌─────────────┴─────────────┐
                 │                           │
          ┌──────┴──────┐            ┌───────┴──────┐
          │  PROD Node  │◄──rsync──►│   DEV Node   │
          │  (Executor) │  15 min    │  (Planner)   │
          └──────┬──────┘            └──────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───┴───┐  ┌────┴───┐  ┌────┴────┐
│ Brain │  │ Router │  │Scheduler│
│(Agent)│  │(Agent) │  │ (Cron)  │
└───┬───┘  └────┬───┘  └────┬────┘
    │           │            │
    └─────┬─────┘      ┌────┴────┐
          │            │ L2 Agents│
    ┌─────┴─────┐      │(Execute)│
    │  External │      └─────────┘
    │   Nodes   │
    └───────────┘
```

## Production Stats

- **Uptime**: Running continuously since March 2026
- **Nodes**: 8 active nodes (2 machines, 15+ agents)
- **Signals processed**: Thousands of JNP messages
- **Communication flows**: 14 defined inter-node flows
- **Cron jobs**: 12+ automated daily operations
- **Recovery events**: Multiple successful failovers between PROD/DEV

## How to Engage

Each RFC ends with **Open Questions** — these are genuine design tensions we haven't fully resolved. We welcome discussion through [GitHub Issues](../../issues) and [Discussions](../../discussions).

If you're building multi-agent systems and hitting similar walls, we'd love to hear your approach.

## License

These design documents are released under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). You're free to share and adapt them with attribution.
