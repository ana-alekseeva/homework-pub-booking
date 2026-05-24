# Ex5 — Edinburgh research loop scenario

## Your answer

Session `sess_44b11c4e303e` (live Nebius run, 2026-05-24T21:07:48) produced
two subgoals. Ticket `tk_22a5f02b` is the planner output (Qwen3-Next-80B-A3B-Thinking,
32 325 ms, 410 tokens in / 5 160 out): sg_1 `"gather all data by calling
venue_search, get_weather, and calculate_cost"` with `assigned_half: "loop"`
and `estimated_tool_calls: 3`; sg_2 `"write the HTML flyer and call
complete_task"` with `assigned_half: "loop"`, `estimated_tool_calls: 2`, and
`depends_on: ["sg_1"]`. The planner assigned both to the loop half because
neither subgoal requires policy enforcement — both are research and output
generation tasks.

Ticket `tk_b06aa2ba` (executor, sg_1, 35 452 ms, 4 turns, 4 tool calls)
ran `venue_search`, `get_weather`, and then `calculate_cost`, followed
prematurely by `complete_task`. The trace shows `venue_search` and
`get_weather` fired in the same executor turn (timestamps
`2026-05-24T21:08:44.452582Z` and `21:08:44.477Z` — 25 ms apart), proving
they ran concurrently: both are `parallel_safe=True` read-only tools.
`calculate_cost(haymarket_tap, 6)` returned `total £556, deposit £111` at
`21:08:50.215848Z`. The executor then called `complete_task` at
`21:09:00.173178Z` without having called `generate_flyer` — sg_1 treated
the gathered data as sufficient and marked the session complete early.

Ticket `tk_0841da16` (executor, sg_2, 46 819 ms, 4 turns, 5 tool calls)
re-ran `venue_search(near="Old Town", party_size=6)` — returning The Royal
Oak — rather than using sg_1's Haymarket Tap result, because the executor
receives only its subgoal description, not sg_1's tool outputs. It then ran
`get_weather` and `calculate_cost(royal_oak, 6)` concurrently
(`21:09:41.635836Z` and `21:09:41.665684Z`, 30 ms apart), obtaining
`total £841, deposit £168`. `generate_flyer` was called at
`21:09:48.177666Z` with the correct key names (`venue_name`, `venue_address`,
`total_gbp`, `deposit_required_gbp`) and wrote `workspace/flyer.html`
(1 026 bytes). `complete_task` followed at `21:09:48.237224Z`.

The dataflow integrity check passed: `verify_dataflow` extracted `£841`,
`£168`, `£556`, `£111`, `12` (temperature), and `cloudy` from the flyer
HTML, and `fact_appears_in_log` found every value in `_TOOL_CALL_LOG`
— `£841` and `£168` in `calculate_cost(royal_oak)`'s output, `12` and
`cloudy` in `get_weather`'s output. The grader's planted value `£9999`
would not appear in any tool record and would return `ok=False` with
`unverified_facts: ["£9999"]`.

## Citations

- `sessions/examples/ex5-edinburgh-research/sess_44b11c4e303e/logs/tickets/tk_22a5f02b/manifest.json` — planner, 2 subgoals, Qwen3-Next-80B-A3B-Thinking
- `sessions/examples/ex5-edinburgh-research/sess_44b11c4e303e/logs/tickets/tk_b06aa2ba/manifest.json` — executor sg_1, 4 tool calls, 4 turns
- `sessions/examples/ex5-edinburgh-research/sess_44b11c4e303e/logs/tickets/tk_0841da16/manifest.json` — executor sg_2, 5 tool calls, 4 turns
- `sessions/examples/ex5-edinburgh-research/sess_44b11c4e303e/logs/trace.jsonl` — parallel venue_search+get_weather at 21:08:44Z; calculate_cost £556/£111; generate_flyer 1026 chars
- `sessions/examples/ex5-edinburgh-research/sess_44b11c4e303e/workspace/flyer.html` — The Royal Oak, £841, £168, cloudy, 12°C
