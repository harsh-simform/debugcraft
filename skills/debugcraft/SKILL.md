---
name: debugcraft
description: >
  Multi-agent root-cause debugging for any incident/anomaly (CPU, latency,
  errors, saturation, cost spike, etc.) in any project, using whatever
  observability MCP servers are connected (Datadog, Azure Monitor/App
  Insights, or others) — never assumes a specific stack. Runs a supervisor-
  directed pipeline (find the window → find the exact cause → prove the
  causal chain, with a loop-back if disproven → identify who/what triggered
  it → parallel adversarial verification → compact plain-English HTML
  report), with token-optimized context handoff and durable error handling.
  Trigger on "/debugcraft", "debug this incident", "root cause this", "why
  did X spike", "investigate this outage", "what caused [metric] to
  [go up/down/fail]".
---

# debugcraft

Seven-agent, supervisor-directed pipeline for going from "something looked wrong" to a proven root cause plus a report a non-engineer can read in three minutes. Project-agnostic: works against whatever observability MCP servers this session has connected — never hardcode a platform name into your reasoning, discover what's available and use it.

Before running a phase, load the two reference docs once at the start of the investigation and keep them in mind for the whole run:
- `references/handoff-schema.md` — the compact envelope every phase returns, and the scratch-file convention (read this first, it governs every prompt you write)
- `references/error-handling.md` — retry ceilings, status semantics, loop-back limit, fan-out partial-failure handling

## Architecture

```
 User: "why did X spike / break / cost more?"
        │
        ▼
 Phase 0 — Scope (orchestrator, no subagent)
 Pin down resource + rough time range. ToolSearch for observability MCP
 tools. Create <scratchpad>/debugcraft-findings.md. If a needed server is
 unreachable, say so and offer retry — don't proceed silently on a gap.
        │
        ▼
 Phase 1 — dbg-signal-hunter          WHEN/WHERE, cross-platform
        │
        ▼
 Phase 2 — dbg-query-profiler         WHAT, ranked by total cost
        │
        ▼
 Phase 3 — dbg-trace-tracer           HOW (proven via correlation ID)
        │
        ├──── route_back_to: 2 ────┐  (at most once — see error-handling.md)
        │                          ▼
        │                    back to Phase 2, re-rank excluding
        │                    the disproven candidate, then Phase 3 again
        ▼
 Phase 4 — dbg-source-identifier      WHO/WHAT triggered it
        │
        ▼
 Phase 5 — parallel fan-out (single Agent-tool message, multiple calls)
   ├── dbg-theory-checker × N   (one call per alternate theory)
   └── dbg-cooldown-analyst × 1 (exact end-of-incident + operational cause)
   Orchestrator collects all verdicts into one scorecard — no extra
   synthesis agent needed, this is cheap mechanical merging.
        │
        ▼
 Phase 6 — dbg-report-writer          compact plain-English HTML, saved
```

Phases 1→4 are a strict pipeline — each needs the previous phase's proven output, so they run one at a time. Phase 5 is the one true fan-out point: every theory check and the cooldown check depend only on phases 1-4's findings, not on each other, so dispatch them together in a single message with multiple `Agent` tool calls (parallel), not one-by-one.

## Running it — context and token rules

- **The scratch file is the source of truth, not the prompt.** After each phase, append its full envelope to `<scratchpad>/debugcraft-findings.md`. When writing the next phase's prompt, pass a one-paragraph synopsis plus the scratch file's path — not the full accumulated history. Full shape and rationale: `references/handoff-schema.md`.
- **Subagents return the compact envelope, never a raw tool dump.** A phase can run as many tool calls as it needs internally (that context is disposable); what comes back to you is the small structured envelope. This keeps your own context flat across the whole run instead of growing with every phase.
- **Never re-derive what a prior phase already found.** If phase 2 already ruled out three candidates, phase 5's theory-checkers should be told that directly (via the synopsis) rather than re-querying to rediscover it.
- **Give each phase the minimum it needs, not everything you have.** More context is not automatically better — a phase 5 theory-checker verifying "was it a bulk upload job" needs the relevant log-search leads, not phase 1's full metric cross-check numbers. Pointing it at the scratch file covers the rare case it needs more.

## Running it — durable error handling (full detail: `references/error-handling.md`)

- One retry per failed tool/MCP call, ever. After that, record the gap (`status: degraded` or `blocked`, `not_obtained` filled in) and move on — don't loop chasing certainty a broken connection can't give you.
- An MCP server that's configured but unreachable is a connection failure, not "unavailable" — say so, offer retry, never conclude the capability doesn't exist.
- Phase 3's loop-back to phase 2 fires at most once per run. If a second pass still can't prove a candidate, proceed with the best inferred one, flagged `confidence: inferred` everywhere downstream and called out in the final report.
- Phase 5's fan-out tolerates individual branch failures — one failed theory-check doesn't block the others or phase 6; it just shows up as `Unresolved: check failed` in the scorecard.
- Escalate to the user (don't silently work around) only for: a whole platform unreachable after retry, the loop-back limit hit with no proven candidate, or a `blocked` phase with no substitute data source.

## When to skip phases

- Already know the exact window? Skip Phase 1, start Phase 2 with the known window in the synopsis.
- Already have the expensive query/endpoint from a monitor/alert? Skip Phase 2, start at Phase 3.
- User only wants a technical answer in chat, no stakeholder report? Skip Phase 6, or offer it as a follow-up.
- Obvious single-cause incident, no live debate about alternatives? Phase 5's theory-checker fan-out can be a small N (1-2 sanity checks instead of a wide sweep) — still always run `dbg-cooldown-analyst`, it's cheap and often surfaces something (an unnoticed deploy).

Never skip Phase 3 once you have a leading cause you're about to name in a report — that's the step that catches a wrong attribution (via correlation-ID proof, not narrative) before it reaches a stakeholder.

## Report defaults

Ask where to save the report if the user didn't say (Desktop is a common default, don't assume). Compact HTML with diagrams (per `dbg-report-writer`) is the default output format unless the user asks for something else (Slack message, doc, plain markdown).

See also: repo `README.md` for the install steps and a Mermaid rendering of this same pipeline.
