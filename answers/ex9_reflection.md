# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session `sess_35e7d93869a4` (Ex7, 2026-05-24), ticket `tk_78fc7bd0`
records the round-1 planner decision. Its `raw_output.json` contains one
subgoal:

```json
{
  "id": "sg_1",
  "description": "find venue near haymarket for 12",
  "assigned_half": "loop",
  "estimated_tool_calls": 2
}
```

The planner assigned this to the loop half because the subgoal is research
— open-ended search with no deterministic acceptance rule. The executor
(ticket `tk_81917cd6`) then ran `venue_search(near="Haymarket",
party_size=12)` at `2026-05-24T21:19:45.263528Z` which returned 0 results
(haymarket_tap has 8 seats, below 12), and then called
`handoff_to_structured` at `21:19:45.283226Z` with `reason: "loop half
identified a candidate venue; passing to structured half for confirmation
under policy rules"` and `data.party_size: "12"`.

The signal that caused the handoff was not the planner's subgoal assignment
— that was `loop` — but the executor completing its research and invoking
the `handoff_to_structured` tool explicitly. The structured half's role is
to apply deterministic policy (party-size cap, deposit ceiling); the loop
half cannot self-certify compliance because its LLM cannot reliably evaluate
hard numeric constraints. After the structured half rejected with
`rejection_reason: "sorry, we can't accept this booking. reason:
party_too_large"`, the bridge fed that string as the next task preview. The
round-2 planner ticket `tk_c0e9c617` received exactly:
`"The structured half rejected the previous proposal. Reason: sorry, we
can't accept this booking. reason: party_too_large. Produce an alternative."`
This injected context is the concrete signal that drove the retry — not a
planner heuristic, but the bridge feeding the rejection back as the next task.

---

## Q2 — Dataflow integrity catch

### Your answer

In session `sess_44b11c4e303e` (Ex5, 2026-05-24), `calculate_cost(royal_oak,
6, 3, bar_snacks)` logged `total_gbp: 841` and `deposit_required_gbp: 168`
in `_TOOL_CALL_LOG` (trace event at `2026-05-24T21:09:41.665684Z`:
`"summary": "calculate_cost(royal_oak, 6): total £841, deposit £168"`).
`generate_flyer` was then called at `21:09:48.177666Z` with
`event_details: {total_gbp: 841, deposit_required_gbp: 168}` and wrote
`workspace/flyer.html` containing `<dd data-testid="total_gbp">£841</dd>`
and `<dd data-testid="deposit_required_gbp">£168</dd>`.

`verify_dataflow` extracts all `£<N>` values from the HTML (after stripping
tags) and calls `fact_appears_in_log` for each. `£841` matches `841` in
`calculate_cost`'s output dict; `£168` matches `168` in the same record;
`12` (temperature) matches `get_weather`'s output; `cloudy` matches the
`condition` field. The function returned `ok=True` with 5 verified facts.

If the LLM had instead written `£9999` — a value absent from every tool
output and every tool argument in `_TOOL_CALL_LOG` — `fact_appears_in_log`
would recursively scan all `.output` and `.arguments` dicts across every
`ToolCallRecord`, find no match, and `verify_dataflow` would return
`ok=False` with `unverified_facts: ["£9999"]`. A manual reviewer looking at
a well-formatted HTML flyer would not independently cross-reference the £
figure against the tool log; the integrity check catches it because the scan
is exhaustive and automatic.

---

## Q3 — First production failure and surfacing primitive

### Your answer

The first failure I would expect in production is a hung LLM call leaving a
ticket permanently in `in_progress`. The Nebius API (or any hosted LLM
endpoint) has variable tail latency: most calls return in two to five seconds,
but occasionally a connection stalls — TCP keep-alive drops, the upstream
worker crashes mid-generation, or a rate-limit queue adds minutes of delay
without signalling failure. When this happens the executor's
`executor.run_subgoal` ticket (e.g. `tk_b06aa2ba` in session
`sess_44b11c4e303e`, `duration_ms: 35452` in the happy path) does not receive
a completion event. It stays in state `in_progress` indefinitely.

The **ticket state machine** is the primitive that surfaces this. Every ticket
has a `started_at` timestamp in `state.json` and exactly one of three terminal
states: `success`, `failure`, or `cancelled`. An operations monitor querying
all tickets with `state == "in_progress"` and `started_at` older than a
configurable deadline can page on-call or trigger a retry. Without the ticket
state machine this failure is invisible: the session directory exists, no
exception was raised, the trace has a `planner.produced_subgoals` event but no
following `executor.tool_called` event — a gap that requires reading two log
files and knowing what sequence to expect. With the state machine, the stale
`in_progress` record is a self-describing, queryable artifact. The ticket ID
and its `operation` field (`executor.run_subgoal/sg_1`) immediately identify
which subgoal hung and in which session, without reconstructing anything from
the raw trace.

---
