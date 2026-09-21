# debugcraft

Multi-agent root-cause debugging plugin for Claude Code. Project-agnostic — works against whatever observability MCP servers a project has connected (Datadog, Azure Monitor/App Insights, or others), on any codebase.

Given an incident/anomaly ("why did staging DB CPU spike yesterday", "why did checkout latency jump", "why did our cloud bill spike"), it runs a supervisor-directed, 7-agent pipeline to go from symptom to proven root cause, then writes a compact, plain-English HTML report with diagrams — the kind a non-technical stakeholder can read in under 3 minutes.

## Install (one-time, per machine)

```
/plugin marketplace add harsh-simform/debugcraft
/plugin install debugcraft@debugcraft-marketplace
```

## Use

Trigger with `/debugcraft`, or just describe the problem: "debug why X spiked", "root cause this outage", "investigate this incident".

## Pipeline

| Phase | Agent                   | Answers                                                                   | Runs                                                                        |
| ----- | ----------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| 1     | `dbg-signal-hunter`     | When and where did it happen, how bad?                                    | sequential                                                                  |
| 2     | `dbg-query-profiler`    | What exactly is consuming the resource?                                   | sequential                                                                  |
| 3     | `dbg-trace-tracer`      | What actually caused it — proven via correlation ID, never time-proximity | sequential, may loop back to phase 2 **once** if its candidate is disproven |
| 4     | `dbg-source-identifier` | Who or what triggered it?                                                 | sequential                                                                  |
| 5a    | `dbg-theory-checker`    | Is alternate theory N confirmed or ruled out?                             | **parallel** — one call per theory                                          |
| 5b    | `dbg-cooldown-analyst`  | Exact end of incident + operational cause?                                | **parallel**, same batch as 5a                                              |
| 6     | `dbg-report-writer`     | Compact HTML report for a stakeholder                                     | sequential, after 5a/5b collected                                           |

Full architecture, context/token rules, and phase-skip guidance: `skills/debugcraft/SKILL.md`.
Handoff envelope shape and scratch-file convention: `skills/debugcraft/references/handoff-schema.md`.
Retry ceilings, degraded/blocked semantics, loop-back and fan-out failure handling: `skills/debugcraft/references/error-handling.md`.

## Workflow

```mermaid
flowchart TD
    U["User: why did X spike / break / cost more?"] --> P0["Phase 0 — Scope<br/>orchestrator: pin resource + window,<br/>discover MCP tools, open scratch file"]
    P0 --> P1["Phase 1 — dbg-signal-hunter<br/>WHEN / WHERE, cross-platform"]
    P1 --> P2["Phase 2 — dbg-query-profiler<br/>WHAT, ranked by total cost"]
    P2 --> P3{"Phase 3 — dbg-trace-tracer<br/>proven via correlation ID"}
    P3 -- "candidate disproven (max once)" --> P2
    P3 -- "proven or inferred" --> P4["Phase 4 — dbg-source-identifier<br/>WHO / WHAT triggered it"]
    P4 --> P5A["Phase 5a — dbg-theory-checker (x N, parallel)<br/>one call per alternate theory"]
    P4 --> P5B["Phase 5b — dbg-cooldown-analyst (parallel)<br/>exact end + operational cause"]
    P5A --> MERGE["orchestrator merges scorecard<br/>(no extra agent needed)"]
    P5B --> MERGE
    MERGE --> P6["Phase 6 — dbg-report-writer<br/>compact plain-English HTML"]
    P6 --> OUT["Saved report + proven root cause"]

    classDef seq fill:#eaf1fd,stroke:#2563eb,color:#1a1d23;
    classDef gate fill:#fbeaea,stroke:#d64545,color:#1a1d23;
    classDef fanout fill:#e6f6ef,stroke:#1a8a5f,color:#1a1d23;
    classDef term fill:#f1f2f5,stroke:#5b6472,color:#1a1d23;
    class P1,P2,P4,P6 seq;
    class P3 gate;
    class P5A,P5B,MERGE fanout;
    class U,P0,OUT term;
```

Phases 1→4 are a strict pipeline (each needs the previous phase's proven output). Phase 3 is the only loop-back point (max once, see error-handling.md). Phase 5 is the only fan-out point — every theory check and the cooldown check are independent of each other, so they dispatch together in one message with multiple parallel `Agent` calls instead of running one at a time.

## CI/CD

There is no central "publish" API for Claude Code plugins — this repo's `.claude-plugin/marketplace.json` is the distribution point itself (see [Install](#install-one-time-per-machine)). `.github/workflows/plugin-ci.yml` covers what CI actually can do:

- **On every push/PR to `main`**: `claude plugin validate` runs `--strict` against `plugin.json` and `marketplace.json`, failing the build on schema errors or warnings.
- **On push to `main`**: if `plugin.json`'s `version` isn't already tagged, it tags the release (`claude plugin tag`) and creates a GitHub Release.

To release a new version: bump `version` in `.claude-plugin/plugin.json`, merge to `main`, CI tags and releases it automatically.

## Design principles

- No assumptions — every claim traces to a tool call.
- Cross-validate across platforms when more than one can see the same fact.
- Time proximity is never treated as proof of causation — only correlation IDs, enforced by phase 3's loop-back authority.
- Self-correct out loud when a later phase disproves an earlier guess, rather than quietly patching over it.
- Ruled-out theories are reported with evidence, not silently dropped.
- Context is passed as compact structured envelopes plus a shared scratch file, not accumulated raw history — keeps the orchestrator's context flat regardless of investigation depth.
- Every tool/MCP failure gets exactly one retry, then becomes a recorded gap (`degraded`/`blocked`), never a silent fabrication or an infinite retry loop.
- The report is compact and plain-English by default — it's for someone who wasn't in the investigation.
