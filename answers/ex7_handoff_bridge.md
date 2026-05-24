# Ex7 — Handoff bridge

## Your answer

Session `sess_35e7d93869a4` (offline FakeLLMClient run, 2026-05-24T21:19:45)
demonstrates the full two-round bidirectional handoff. The trace contains
four `session.state_changed` events:

```
round 1: loop → structured  (2026-05-24T21:19:45Z)
round 1: structured → loop  (rejection_reason: "sorry, we can't accept this booking. reason: party_too_large")
round 2: loop → structured  (2026-05-24T21:19:46Z)
round 2: structured → complete
```

Round 1: ticket `tk_78fc7bd0` (planner, 16 ms) produced one subgoal —
`"find venue near haymarket for 12"` with `assigned_half: "loop"`. Ticket
`tk_81917cd6` (executor, sg_1, 46 ms, 2 turns) ran
`venue_search(near="Haymarket", party_size=12, budget_max_gbp=2000)` at
`21:19:45.263528Z` which returned 0 results (haymarket_tap has only 8 seats,
below the requested 12), then called `handoff_to_structured` at
`21:19:45.283226Z` with `reason: "loop half identified a candidate venue;
passing to structured half for confirmation under policy rules"` and
`data.party_size: "12"`. The structured half rejected because party 12
exceeds the 8-seat cap, writing `rejection_reason: "sorry, we can't accept
this booking. reason: party_too_large"` back to the bridge.

Round 2: ticket `tk_c0e9c617` (planner, 15 ms) received the rejection
injected as its task preview — `"The structured half rejected the previous
proposal. Reason: sorry, we can't accept this booking. reason:
party_too_large. Produce an alternative."` — and produced a retry subgoal
`"retry with larger venue after rejection"`. Ticket `tk_78cce549` (executor,
sg_1, 47 ms, 2 turns) ran `venue_search(near="Old Town", party_size=6,
budget_max_gbp=2000)` at `21:19:46.324367Z` — 1 result (The Royal Oak,
16 seats) — then called `handoff_to_structured` at `21:19:46.345683Z`
with `reason: "retry after reverse handoff — scaled down to fit policy"` and
`data: {venue_id: "The Royal Oak", party_size: "6", deposit: "£0"}`. The
structured half confirmed. The bridge transitioned to `complete`.

The IPC file is overwritten atomically on each round, so at most one
handoff file is visible in `ipc/` at any time — the fail-closed rule is
satisfied. The previous round's payload is not preserved in `ipc/`, but
each ticket's `raw_output.json` retains it via `handoff_payload` for the
audit trail.

## Citations

- `sessions/examples/ex7-handoff-bridge/sess_35e7d93869a4/logs/trace.jsonl` — four state_changed events across two rounds
- `sessions/examples/ex7-handoff-bridge/sess_35e7d93869a4/logs/tickets/tk_78fc7bd0/raw_output.json` — round 1 planner subgoal (assigned_half: loop)
- `sessions/examples/ex7-handoff-bridge/sess_35e7d93869a4/logs/tickets/tk_81917cd6/raw_output.json` — round 1 executor, handoff_payload with party_size 12
- `sessions/examples/ex7-handoff-bridge/sess_35e7d93869a4/logs/tickets/tk_c0e9c617/raw_output.json` — round 2 planner, rejection injected as task preview
- `sessions/examples/ex7-handoff-bridge/sess_35e7d93869a4/logs/tickets/tk_78cce549/raw_output.json` — round 2 executor, venue_search Old Town + handoff party_size 6
