# RFC-001: JNP — Asynchronous Inter-Agent Communication Protocol

> **Status**: Draft
> **Author**: AetherOS Team
> **Created**: 2026-03-26
> **Version**: Based on JNP v1.1 production specification

## Abstract

The Joint Nerve Protocol (JNP) is an asynchronous, file-based communication protocol designed for coordinating AI agent clusters running on local infrastructure. Unlike synchronous RPC or API-based agent communication, JNP treats inter-agent messages as **self-describing documents** that can be stored, forwarded, audited, and replayed — making the entire communication history inspectable and recoverable.

This RFC describes the protocol design, the reasoning behind key decisions, and lessons learned from months of production use.

---

## 1. The Problem

### 1.1 Why existing approaches fall short

Most multi-agent frameworks communicate through synchronous function calls or API requests:

```
Agent A  ──call──►  Agent B  ──response──►  Agent A
```

This works for demos and simple chains. It breaks in production because:

- **Coupling**: If Agent B is down, Agent A blocks. In a system with 15+ agents across multiple machines, something is always down.
- **No audit trail**: Function calls vanish after execution. When something goes wrong at 3 AM, there's nothing to inspect.
- **No priority**: A routine status update and a critical security alert use the same channel. There's no way to express urgency.
- **No partial failure**: If the network drops mid-response, both sides may have inconsistent state.
- **No human interoperability**: Humans can't "read" a function call or inject a message into the system without writing code.

### 1.2 What we actually need

A production agent communication system needs:

1. **Asynchronous delivery** — Send and forget. The recipient processes when ready.
2. **Self-describing messages** — Any message should be understandable without external context.
3. **Priority signaling** — Critical alerts must be distinguishable from routine updates.
4. **Type orthogonality** — "What kind of action" and "how urgent" are independent dimensions.
5. **Auditability** — Every message persists. The full history is always available for debugging.
6. **Human-readable format** — Operators should be able to read, write, and modify messages directly.
7. **Zero external dependencies** — No database, no message queue, no cloud service required.

---

## 2. Design Decisions

### 2.1 Messages are files

The most controversial decision: every JNP message is a Markdown file with YAML frontmatter.

```markdown
---
msg_id: "20260320-1430-node-alpha-P1-deploy-report"
author: node-alpha
ts: "2026-03-20T14:30:00+08:00"
priority: P1
type: result
to: [node-beta]
ref: "20260320-0900-node-beta-P1-deploy-task"
---

# Deployment Report

- Status: Success
- Duration: 12 minutes
- Artifacts: 3 files updated
```

**Why files?**

- **Debuggable**: `ls inbox/` tells you the system state. `cat` any message to read it. No special tools needed.
- **Recoverable**: Files survive process crashes, agent restarts, and even machine reboots. They're backed up by normal filesystem tools.
- **Transportable**: `rsync` moves files between machines. No custom transport protocol needed.
- **Versionable**: `git` can track message history if needed.
- **Human-writable**: An operator can create a JNP message with any text editor.

**The tradeoff**: File I/O is slower than in-memory message passing. For our workload (hundreds of messages per day, not thousands per second), this is irrelevant. If you're building a high-frequency trading system, JNP is not for you.

### 2.2 Type × Priority orthogonality

JNP defines two independent axes for every message:

**Signal Types** (what to do with it):

| Type | Semantics | Expected Response |
|------|-----------|-------------------|
| `task` | Work delegation | Execute, return `result` |
| `result` | Task completion report | Originator reads and closes |
| `alert` | Urgent notification | Immediate action required |
| `decision` | Policy or rule change | Update local configuration |
| `sync` | State synchronization | Update local cache |
| `fyi` | Informational notice | Archive, no action needed |

**Priority Levels** (when to handle it):

| Priority | Deadline | ACK Required |
|----------|----------|--------------|
| `P0` | Immediate | Yes, within 30 min |
| `P1` | Within 24h | Task types: yes. Others: optional |
| `P2` | This week | No |
| `FYI` | No deadline | No |

**Why orthogonal?** Because a `task` can be urgent (`P0`: "the server is down, fix it now") or routine (`P2`: "when you have time, update the docs"). Conflating type with priority forces artificial categories like "critical-task" vs "normal-task" — a combinatorial explosion that gets worse with every new type or priority level.

With orthogonality, adding a new type doesn't require defining new priorities, and vice versa. The system scales linearly, not quadratically.

### 2.3 Filename as metadata

JNP message filenames encode key metadata:

```
YYYYMMDD-HHMM-{author}-{priority}-{slug}.md
20260320-1430-node-alpha-P1-deploy-report.md
```

**Why?** Because `ls -lt inbox/` becomes a dashboard:

```
20260320-1430-node-alpha-P0-gateway-down.md    ← Critical alert
20260320-1420-node-beta-P1-test-result.md      ← Test completed
20260320-1400-node-cron-FYI-cron-ok.md         ← Routine update
```

Without opening any file, you can see: who sent it, when, how urgent, and roughly what it's about. This is invaluable during incident response.

### 2.4 ACK mechanism with auto-escalation

Not all messages need acknowledgment. The ACK rules follow a simple principle: **the higher the priority, the stricter the follow-up**.

| Scenario | Rule |
|----------|------|
| Any P0 | Must ACK within 30 min via `{slug}-ack.md` |
| P1 + task | Writing `type: result` counts as ACK |
| P1 + decision | ACK if actionable |
| P2, FYI | No ACK needed |

**Auto-escalation on timeout**:
- P0 unacknowledged after 30 min → alert broadcast to all nodes + human notification
- P1 task unacknowledged after 4h → alert to originator + included in daily report

This prevents the "message sent into void" problem without requiring every message to be explicitly acknowledged.

### 2.5 Idempotency (v1.1)

In a system with periodic `rsync` between machines, the same file may arrive multiple times. JNP v1.1 introduced idempotency keys:

```yaml
msg_id: "20260320-1907-node-beta-FYI-research-result"
idempotency_key: "research-manual-run-1"
```

**Rule**: Same `ref` + same `type` + same `idempotency_key` = last-write-wins.

This ensures that resyncing the inbox doesn't cause duplicate processing.

---

## 3. Routing and Validation

### 3.1 Ingress validation

Before a message enters the routing pipeline, it must pass:

1. **Extension check**: Must be `.md` (other files → `inbox/quarantine/`)
2. **Filename format**: Must match `YYYYMMDD-HHMM-{author}-{priority}-{slug}.md`
3. **Required frontmatter**: `author` + `ts` + `priority` + `type`

Invalid messages are quarantined, not dropped. This preserves debugging information while preventing malformed messages from disrupting the system.

### 3.2 Topology-aware routing

Not every node can talk to every other node. AetherOS defines an explicit communication topology:

```yaml
# allowed-flows.yaml (simplified)
flows:
  - from: node-alpha → to: node-beta       # PROD ↔ DEV bilateral
  - from: node-alpha → to: node-router     # Brain → Router
  - from: node-alpha → to: node-scheduler  # Brain → Scheduler
  # ... 14 flows total
```

Messages addressed to unreachable nodes are rejected at routing time with a clear error, rather than silently disappearing.

**Exception**: P0 messages bypass topology constraints. During a crisis, any node can reach any other node.

### 3.3 Broadcast and wildcard

```yaml
to: []          # Empty = all nodes process
to: [all]       # Explicit broadcast
to: [cluster-b-*]   # Wildcard matching
```

---

## 4. Production Validation

### 4.1 What worked well

- **File-based debugging**: During incidents, `ls -lt inbox/` immediately shows the message flow. No log parsing, no special tools.
- **rsync as transport**: Zero configuration, zero maintenance. Works over SSH, handles interruptions gracefully.
- **Human-in-the-loop messages**: Operators regularly create manual JNP messages to inject instructions into the system. This was not planned but emerged naturally from the file-based design.
- **Cross-machine recovery**: When PROD went down for maintenance, DEV continued processing messages. When PROD came back, `rsync` caught it up. Zero messages lost.

### 4.2 What we learned the hard way

- **Timestamp discipline is critical**: Early versions allowed relative timestamps ("2 hours ago"). This caused confusion when messages were replayed. We now enforce ISO 8601 with explicit timezone (`+08:00`).
- **Slug uniqueness matters**: Two messages with similar slugs caused routing confusion. We added `msg_id` as an explicit unique identifier in v1.1.
- **Archive policy prevents bloat**: Without auto-archiving, the inbox grew to hundreds of files, slowing `ls` and making triage harder. Current policy: >7 days → auto-archive, >90 days → delete.
- **Priority inflation is real**: Early on, everything was P0 or P1. We added P2 in v1.1 and established clear escalation criteria to push back on over-prioritization.

---

## 5. Comparison with Alternatives

| Aspect | JNP (File-based) | Message Queue (RabbitMQ/Redis) | Direct API Calls |
|--------|-------------------|-------------------------------|-----------------|
| Dependencies | None (filesystem) | Queue server required | Network required |
| Debuggability | `ls` + `cat` | Queue-specific tools | Log aggregation |
| Persistence | Automatic (files) | Configurable | None by default |
| Human-readable | Yes (Markdown) | Binary/JSON | Depends |
| Cross-machine | rsync | Network protocol | Network protocol |
| Recovery | File backup | Queue replay | No built-in |
| Throughput | Hundreds/day | Millions/day | Thousands/sec |
| Latency | Seconds (rsync interval) | Milliseconds | Milliseconds |

**JNP is not a general-purpose message queue.** It's designed for agent coordination workloads where debuggability, recoverability, and human interoperability matter more than throughput or latency.

---

## 6. Open Questions

We welcome discussion on these unresolved design tensions:

### 6.1 Scaling beyond two machines

The current rsync-based transport works for 2 machines. With 5+ machines, full-mesh rsync becomes O(n²). Should we introduce a relay topology? A lightweight pub/sub layer? Or is the file-based approach fundamentally limited to small clusters?

### 6.2 Real-time use cases

Some operations (e.g., live user queries) need sub-second response. Currently these bypass JNP entirely and use direct API calls. Should JNP support a "fast path" mode, or is it correct to treat real-time and async as separate systems?

### 6.3 Schema evolution

Adding new fields to frontmatter is easy (backward-compatible). But changing the semantics of existing fields or removing them requires coordinated updates across all nodes. How should protocol versioning work in a system where nodes update asynchronously?

### 6.4 Security model

Currently, any process that can write to `inbox/` can inject messages. There's no authentication or signing. For a local-only system this is acceptable, but if JNP were extended to untrusted networks, what's the minimal security model that preserves simplicity?

### 6.5 Ordering guarantees

JNP provides no message ordering guarantees beyond timestamps. Two messages sent within the same minute may have identical filename prefixes. Is timestamp-based ordering sufficient, or do we need sequence numbers?

---

## Appendix A: Full Frontmatter Schema

### Required fields (all messages)

```yaml
author: string       # Node ID of sender (e.g., "node-alpha")
ts: string           # ISO 8601 timestamp with timezone (e.g., "2026-03-20T14:30:00+08:00")
priority: enum       # P0 | P1 | P2 | FYI
type: enum           # task | result | alert | decision | sync | fyi
```

### Optional fields

```yaml
msg_id: string       # Unique message identifier
to: [string]         # Recipient node IDs (empty = broadcast)
ref: string          # Reference to related message (required for type: result)
affects: [string]    # Impact scope description
deadline: string     # ISO 8601 deadline (for tasks)
needs_ack: boolean   # Explicit ACK request
status: enum         # unread | acked | in_progress | done | dropped
owner: string        # Responsible node
supersedes: string   # ID of message this replaces
idempotency_key: string  # Dedup key for replayed messages
```

---

*This RFC is part of the [AetherOS Design RFCs](README.md) series. Discussion welcome via [GitHub Issues](../../issues).*
