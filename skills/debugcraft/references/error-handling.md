# Durable error handling

Every phase and the orchestrator follow the same failure playbook. Failures are data, not stop conditions — a phase that hits an error still returns a valid envelope, just with `status: degraded` or `status: blocked` and an honest `not_obtained` list. The investigation never silently fabricates to paper over a gap, and never crashes the whole pipeline over one bad tool call.

## Tool/MCP failure classes and responses

| Failure | Detection | Response |
|---|---|---|
| MCP server not connected / deferred tool not found | ToolSearch returns nothing, or a tool call errors with connection/timeout text | Treat as a connection failure, not "capability doesn't exist" (this is explicit in the platform's own tool-search guidance). Retry the ToolSearch/tool call **once**. Still failing → mark that data source `status: blocked` in the envelope, name the server, tell the orchestrator to surface it to the user (fixing an MCP connection is a user action, not something a subagent can do). |
| Query/call times out | Explicit timeout error, or no response within the tool's own limit | Retry **once** with a narrower scope (shorter time range, smaller result limit) — many timeouts are size-driven, not availability-driven. Still failing → `status: degraded`, note in `not_obtained`. |
| Empty result | Tool succeeds, returns zero rows/no match | Not an error. Record as a real negative finding ("searched X, zero hits") — this is exactly the evidence a ruled-out theory needs. Do not retry a genuinely empty result. |
| Ambiguous match (e.g. 3 resources share a name) | Search returns multiple candidates for one expected identifier | Do not guess. Pick the most contextually likely one, but say so explicitly and name the alternates in `not_obtained`/notes, so a later phase can revisit if the theory doesn't hold up. |
| Rate limit / throttling | 429-style error or explicit rate-limit message | Back off once (do not hammer), retry once. Still failing → `status: degraded`, defer that specific query, continue with what's already gathered. |
| Malformed/unexpected response shape | Tool returns something the phase can't parse as expected | Retry once with a simplified query. Still failing → `status: degraded`, note the exact tool+query that misbehaved so a human can investigate the integration itself. |

**Retry ceiling: one retry per failure, ever.** A phase that keeps retrying the same failing call is burning tokens for no new information. After one retry, the failure becomes a recorded gap, not a loop.

## Phase-level status field (required in every envelope)

- `ok` — everything needed for this phase was obtained.
- `degraded` — phase completed with some gaps (a query failed, a server was unavailable); the phase's conclusion may be weaker than normal, `not_obtained` lists exactly what's missing.
- `blocked` — phase could not produce a usable finding at all (e.g. the only data source for this phase is completely unavailable). The orchestrator must decide: pause and ask the user to fix access, or continue the pipeline with this phase explicitly marked unresolved in the final report.

The orchestrator never treats a `degraded` or `blocked` phase as if it were `ok`. Downstream phases receive the status alongside the findings, and the final report's "still open" section is partly built from every non-`ok` status across the whole run.

## Loop-back cycle limit (phase 3 → phase 2)

Phase 3 (`dbg-trace-tracer`) may route the orchestrator back to phase 2 exactly **once**, only when correlation-ID proof fully invalidates phase 2's candidate (not merely refines it). On the second time a re-run of phase 2/3 still can't produce a proven candidate, stop looping: proceed to phase 4 onward with the best available (unproven) candidate, explicitly flagged `confidence: inferred` in every downstream envelope and called out in the final report's "still open" section. Never loop more than once — ambiguous evidence should surface to the user, not consume the budget searching for certainty that may not exist.

## Fan-out partial failure (phase 5)

Phase 5 dispatches multiple parallel `dbg-theory-checker` calls plus one `dbg-cooldown-analyst` call. If one of these fails (tool error, timeout after one retry) it does not fail the batch — the orchestrator collects whatever verdicts came back, marks the failed one `Unresolved: check failed, needs manual re-run` in the scorecard, and proceeds. Never block phase 6 on a single failed parallel branch when the others succeeded.

## Escalation to the user

Surface to the user, do not silently work around, when:
- An entire observability platform needed for the investigation is unreachable after one retry (the user may need to fix credentials/connectivity).
- Phase 3's loop-back limit is hit without a proven candidate (this is a real ambiguity worth a human decision, not something to force an answer to).
- A phase is `blocked` and no other data source can substitute.

Everything else (single empty results, one retried timeout that then succeeded, an ambiguous-but-resolved match) is handled inline and just shows up as a normal note in the envelope — it does not need to interrupt the user.
