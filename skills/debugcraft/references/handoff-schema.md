# Phase handoff — token-optimized

Two rules override everything below:

1. **The scratch file is the source of truth. The prompt is a pointer, not a payload.** Every phase's full findings get appended to `<scratchpad>/debugcraft-findings.md` (Write/Edit tool). The orchestrator does NOT re-paste every prior phase's full output into each new agent's prompt — that grows the orchestrator's own context linearly (and quadratically across the whole run) for information the next agent may not even need in full. Instead, each subagent prompt gets: the scratch file's path, a **one-paragraph synopsis** (not the full envelope) of only what that phase needs to know, and an instruction to `Read` the scratch file itself if it needs more than the synopsis. This mirrors the standard "just enough context, not exhaustive context" principle — give an agent the minimum it needs to do its job well, let it pull more only if it decides it needs to.
2. **Subagents return a compact envelope to the orchestrator, never a raw tool dump.** A subagent may run a dozen tool calls internally — that's fine, it's happening inside that agent's own disposable context. What comes back to the orchestrator (and what the orchestrator's own context has to hold) is the small structured envelope below. This keeps the orchestrator's context flat and cheap regardless of how much investigation happened inside any one phase.

## Envelope (every phase returns exactly this shape, target under ~250 words in the `findings` fields combined)

```yaml
phase: <1-6>
agent: <agent name>
status: ok | degraded | blocked      # see references/error-handling.md
findings:
  # phase-specific fields, see per-phase shapes below — keep terse, facts only
not_obtained:
  # what you could not get and why, or "none"
route_back_to: null | 2               # only dbg-trace-tracer may set this, and only once per run
confidence: proven | inferred | unresolved
```

The orchestrator appends the full envelope (plus any longer supporting detail the agent wants preserved) to the scratch file under a `## Phase N` heading, and only carries the **synopsis line** forward into the next prompt, e.g.:

> "Phase 2 found query signature `X` responsible for 94% of cost in the window, CPU-bound (0 disk reads), literal SQL captured. Full detail in scratch file under '## Phase 2'."

## Per-phase `findings` shape

```yaml
# Phase 1 — dbg-signal-hunter
resource: <exact name>
window: {start, end, tz}
peak: <value+unit>
baseline: <value+unit>
cross_check: <agree|disagree|single-source>, <system A value> vs <system B value or "n/a">

# Phase 2 — dbg-query-profiler
candidate: <exact identifier>
cost_share: <% of total>
mechanism: io-bound | cpu-bound, <one evidence number>
literal_source: <verbatim, or pointer to scratch file if long>
ruled_out_candidates: <short list with numbers, or "none close">

# Phase 3 — dbg-trace-tracer
proven_parent: <endpoint/service>
proof: <correlation ID field + value/pattern>
correction: <what changed from an earlier guess, or "none">
sampling_caveat: <note, or "none">
route_back_to: null | 2   # set to 2 only if phase-2 candidate is fully invalidated, only once

# Phase 4 — dbg-source-identifier
caller_class: human | script-or-job | internal-service | external-integration
evidence: <short bullets>
best_guess_identity: <name or "unresolved">
call_sequence: <ordered list, terse>

# Phase 5a — dbg-theory-checker (one per theory, dispatched in parallel)
theory: <name>
verdict: confirmed | ruled-out | unresolved
evidence: <one line, what was checked and found>

# Phase 5b — dbg-cooldown-analyst (runs in the same parallel batch as 5a)
cooldown_window: <exact timestamps, before/after values>
operational_cause: <found: what | not found: what was checked>

# Phase 6 — dbg-report-writer
report_path: <where it was saved>
```

## Why this shape

- Fixed field names mean the orchestrator can assemble the final synthesis (theory scorecard, cross-checked numbers table) by reading structured fields, not re-parsing prose.
- A hard word budget on `findings` forces each phase to state its conclusion, not narrate its process — the process (every tool call, every intermediate dead end) lives in the scratch file for audit, not in what gets carried token-for-token through the whole run.
- `not_obtained` and `status` are always present, even when empty/ok — this is what makes gaps visible by construction instead of by remembering to mention them.
