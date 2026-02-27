# nail-sentinel

> AI agents need brakes. nail-sentinel is the brakes.

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![NAIL](https://img.shields.io/badge/powered%20by-NAIL-orange)](https://github.com/watari-ai/nail)

nail-sentinel is a runtime safety layer for AI agents, powered by [NAIL](https://github.com/watari-ai/nail).

Where [mcp-fw](https://github.com/zyom45/mcp-fw) controls *which tools an agent can see*, nail-sentinel controls *what an agent can do while running* — enforcing budgets, invariants, and guaranteed stop signals.

## The Problem

> "I asked my AI to organize my emails. Context compaction dropped the 'organize' part. The agent deleted everything. STOP had no effect."

This failure has three layers:
1. **Intent was lost** — context compaction ate the goal
2. **No quantity limits** — destructive ops ran unbounded  
3. **No interrupt mechanism** — STOP was ignored at runtime

nail-sentinel addresses all three at the language level.

## Core Primitives

### BUDGET — Execution limits
```nail
BUDGET calls=50, mutations=10, time=30s
DELETE_EMAIL(id: STRING) → VOID
effect: IO.destructive
```
Hard stop when limits are reached. The runtime enforces this, not the LLM.

### INVARIANT — State constraints
```nail
INVARIANT: deleted_count <= 5
INVARIANT: total_emails_after >= (total_emails_before * 0.8)
```
Conditions that must hold throughout execution. Violated invariant = immediate halt.

### @interruptible — Guaranteed stop
```nail
@interruptible(checkpoint_every=5)
PROCESS_EMAILS(ids: LIST<EMAIL_ID>) → VOID
```
Every 5 operations, the runtime checks for interrupt signals (SIGINT, Slack STOP, API cancel). Stop is **guaranteed** to be honored.

### @irreversible + dry-run — Destructive op awareness
```nail
@irreversible
DELETE_EMAIL(id: STRING) → VOID
effect: IO.destructive
```
```bash
nail run policy.nail --dry-run
# Lists all @irreversible calls without executing them
```

### SCOPE — Access boundaries
```nail
SCOPE: email/inbox/unread, email/inbox/spam
FORBIDDEN: email/sent, email/drafts
```
Static analysis blocks out-of-scope access before execution.

### @confirm — Human-in-the-loop
```nail
@confirm(level="destructive", channel="slack")
DROP_TABLE(name: STRING) → VOID
```
Requires human approval before executing destructive operations.

## Relationship to mcp-fw

| | mcp-fw | nail-sentinel |
|---|---|---|
| Target | MCP tools (static) | Agent execution (runtime) |
| Controls | Tool availability | Behavior during a run |
| Enforcement | Before invocation | During execution |
| STOP signal | — | Language-level guarantee |

Both tools compose: mcp-fw at the perimeter, nail-sentinel at the runtime.

## Status

🚧 **Design phase** — Primitives are being finalized. Feedback welcome.

- Core NAIL: [watari-ai/nail](https://github.com/watari-ai/nail)
- MCP layer: [zyom45/mcp-fw](https://github.com/zyom45/mcp-fw)

## License

MIT
