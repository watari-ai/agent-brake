# agent-brake

> AI agents need brakes. agent-brake is the brakes.

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![NAIL](https://img.shields.io/badge/powered%20by-NAIL-orange)](https://github.com/watari-ai/nail)

**agent-brake** is a runtime safety layer for AI agents.

It sits between your AI agent and the world, enforcing hard limits that the LLM cannot override — no matter what the conversation context says.

## The Problem

> "I asked my AI to organize my emails. Context compaction dropped the 'organize' part. The agent deleted everything. STOP had no effect."

Three layers of failure:

1. **Intent was lost** — context compaction ate the goal, leaving only "delete"
2. **No quantity limits** — destructive ops ran unbounded
3. **No interrupt mechanism** — STOP messages were ignored at runtime

agent-brake addresses all three at the runtime level.

## Core Primitives

### BUDGET — Hard execution limits
```
BUDGET calls=50, mutations=10, time=30s
```
The runtime enforces this. Not the LLM. When the budget is hit, execution stops unconditionally.

### INVARIANT — State constraints
```
INVARIANT: deleted_count <= 5
INVARIANT: total_emails_after >= (total_emails_before * 0.8)
```
Conditions checked before every destructive operation. Violated = immediate halt.

### @interruptible — Guaranteed STOP
```
@interruptible(checkpoint_every=5)
```
Every 5 operations, the runtime checks for stop signals:
- `SIGINT` (Ctrl+C)
- Message containing "stop", "halt", "やめて", "止まって" etc.
- HTTP `DELETE /run/{id}`

**STOP is always honored within N operations. This is a runtime guarantee, not an LLM promise.**

### @irreversible — Destructive op marking
```
@irreversible
DELETE_EMAIL(id: STRING) → VOID
```
```bash
agent-brake run --dry-run  # lists all @irreversible calls without executing
```

### SCOPE — Access boundaries
```
SCOPE: email/inbox/**
FORBIDDEN: email/sent, email/drafts
```
Static enforcement. Out-of-scope access never reaches the agent.

### @confirm — Human-in-the-loop gate
```
@confirm(level="destructive", channel="slack")
DROP_TABLE(name: STRING) → VOID
```
Requires explicit human approval. Times out and denies if no response.

## Quick Start

```bash
pip install agent-brake  # coming soon
```

```yaml
# brake.yaml
budget:
  calls: 100
  mutations: 20
  time: 5m

invariants:
  - "deleted_count <= 5"
  - "total_items_after >= (total_items_before * 0.80)"

interruptible:
  checkpoint_every: 10
  channels:
    - sigint
    - natural_language:
        patterns: ["stop", "halt", "cancel", "やめて", "止まって", "止めて"]
        confidence_threshold: 0.85
    - api

scope:
  allow: ["email/inbox/**"]
  deny: ["email/sent", "email/drafts"]

confirm:
  destructive:
    channel: slack
    timeout: 5m
    default: deny
```

## Composing with mcp-fw

agent-brake works at the **runtime layer**, complementing [mcp-fw](https://github.com/zyom45/mcp-fw) which works at the **MCP tool layer**:

```
AI Agent
    │
    ▼
┌───────────┐
│  mcp-fw   │  ← controls which tools the agent can see (static)
└─────┬─────┘
      │
      ▼
┌───────────────┐
│  agent-brake  │  ← controls what the agent does while running (dynamic)
│               │    BUDGET / INVARIANT / STOP signal / SCOPE
└───────────────┘
```

Use both for defense in depth.

## Powered by NAIL

agent-brake uses [NAIL](https://github.com/watari-ai/nail)'s effect type system to classify operations automatically. You don't need to manually tag every function — NAIL infers whether a call reads, writes, deletes, or accesses the network.

## Status

🚧 **Design phase** — Runtime implementation in progress.

## License

MIT
