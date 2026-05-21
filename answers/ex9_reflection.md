# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session sess_da3622747b05, the planner produced two subgoals and assigned both to the loop half (sg_1: research a suitable venue; sg_2: draft the flyer) — assigned_half: "loop" in each. The planner did not route anything to the structured half. The handoff instead arose at execution time, inside sg_1.

While executing sg_1, the executor called venue_search four times (Old Town, Grassmarket, Edinburgh ×2), every call with party_size: 50 and budgets rising 500→1500 GBP, and every call returned 0 result(s). The fourth call tripped the spiral guard ("STOP calling venue_search; use the results you already have."). The executor then emitted a handoff_to_structured call with reason: "No venues found after multiple searches" and context: "Tried areas: Old Town, Grassmarket, Edinburgh with budgets 500-1500 GBP for party of 50"; the ticket records handoff_requested: true, search_attempts: 4, turns_used: 5.

The signal driving the handoff was therefore not the planner's prose interpretation but an empirical dead-end at runtime: repeated tool failure plus the spiral-guard stop condition. The lesson — escalate to a different half when the current half exhausts its options, rather than looping indefinitely.

Notably, the planner did not route it to the structured half. The planner assigned both subgoals to the loop half (assigned_half: "loop"). The handoff instead emerged at execution time, when the executor exhausted its options. This is itself the answer to 'where does the handoff decision live'. In this architecture, it is not always a planner-level routing choice, it could be a runtime escalation. The planner's decision was to not pre-route, trusting the executor to escalate if research failed, which is exactly what happened. 

### Citation

- sessions/sess_da3622747b05/logs/tickets/tk_9d343cc4/raw_output.json
- sessions/sess_da3622747b05/logs/tickets/tk_9642df2f/raw_output.json

---

## Q2 — Dataflow integrity catch

### Your answer

My integrity check `verify_dataflow` (starter/edinburgh_research/integrity.py)
catches fabrications by recomputing every fact in the flyer against
`_TOOL_CALL_LOG` — the record of what tools actually returned — rather than
trusting the flyer's text. A fact that appears in the flyer but in no tool
output is flagged as unverified and the check returns ok=False.

The grader's dataflow probe demonstrates this concretely. It plants three
fabrications into a flyer and confirms my check catches all of them
(score 6/6): an impossible price not produced by any venue+catering
combination (£9999); a non-existent venue name ("Castle Royal Grand Inn")
that no `venue_search` call ever returned; and an impossible Edinburgh
temperature ("scorching 35C") when `get_weather` returned no such value.

The temperature case is the clearest. In a legitimate offline run
(`make ex5`), `get_weather` returns condition "cloudy" and temperature 12,
and the flyer carries those exact values — `verify_dataflow` confirms all
10 facts and returns ok=True. But if the LLM had written "35C" or
"scorching", the check recomputes against the log, finds no tool call
produced 35, and fails: unverified_facts would contain the fabricated
temperature. A human skimming the flyer would not notice 35 is wrong;
the check does, because it compares against ground truth in the tool log,
not against "does this look plausible."

The lesson: validation must verify provenance (where did this value come
from?), not mere presence (is a number here?). My §2.7 extension hardens
this by also catching label-colon and adjective-temperature patterns that
the shipped substring check missed.

### Citation

- starter/edinburgh_research/integrity.py — verify_dataflow recomputes facts vs _TOOL_CALL_LOG
- grader/dataflow_probe.py — plants £9999, "Castle Royal Grand Inn", "scorching 35C"; my check catches all 3 (score 6/6)

---

## Q3 — Removing one framework primitive

### Your answer

**Primitive:** the dataflow integrity check (`verify_dataflow` in
starter/edinburgh_research/integrity.py).

**Failure mode:** self-verifying validation — a validator that reads from
the same state the artefact-producing tool wrote into, so it confirms a
value's presence rather than its provenance.

The shipped `fact_appears_in_log` scans `_TOOL_CALL_LOG` for a fact's
existence anywhere in the log — including the arguments of the very call
that produced the artefact. When the agent calls
`generate_flyer(total_gbp=540, ...)`, that 540 is written into the call's
arguments, and the later check finds 540 in the log and confirms it. The
flyer becomes the witness for its own claim. The validator is technically
functioning — the value it seeks is present — but it is not checking
against an independent source of truth. The check passes on a fabricated
number because the fabrication wrote itself into the state being scanned.

This is a real production pattern, not a toy bug. The same shape appears
when a compliance log validator reconstructs events from the very log it
is meant to verify, or when a pricing audit re-reads the figure the
pricing engine just emitted — the audit "passes" because the artefact and
its check share state.

The fix is to break the shared path: recompute the value from primary
inputs inside the validator (venue rates and party size for cost), and
compare against what the flyer claims, failing loudly on divergence. That
is the difference between asking "is this value present?" and "where did
this value come from?" — validation must answer the second. My 2.7
extension moves toward this by recomputing facts against tool *outputs*
rather than trusting the artefact's own text.

### Citation

- starter/edinburgh_research/integrity.py — verify_dataflow / fact_appears_in_log shared-state check
