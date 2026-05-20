# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session sess_da3622747b05, the planner produced two subgoals and assigned both to the loop half (sg_1: research a suitable venue; sg_2: draft the flyer) — assigned_half: "loop" in each. The planner did not route anything to the structured half. The handoff instead arose at execution time, inside sg_1.

While executing sg_1, the executor called venue_search four times (Old Town, Grassmarket, Edinburgh ×2), every call with party_size: 50 and budgets rising 500→1500 GBP, and every call returned 0 result(s). The fourth call tripped the spiral guard ("STOP calling venue_search; use the results you already have."). The executor then emitted a handoff_to_structured call with reason: "No venues found after multiple searches" and context: "Tried areas: Old Town, Grassmarket, Edinburgh with budgets 500-1500 GBP for party of 50"; the ticket records handoff_requested: true, search_attempts: 4, turns_used: 5.

The signal driving the handoff was therefore not the planner's prose interpretation but an empirical dead-end at runtime: repeated tool failure plus the spiral-guard stop condition. The lesson — escalate to a different half when the current half exhausts its options, rather than looping indefinitely.

### Citation

- ~/.local/share/sovereign-agent/examples/ex5-edinburgh-research/sess_da3622747b05/logs/tickets/tk_9d343cc4/raw_output.json

---

## Q2 — Dataflow integrity catch

### Your answer

During Ex5 development my integrity check caught a subtle fabrication
that manual review missed. In session sess_de44a1b8eb12 the flyer
claimed "Total: £560" and "Deposit: £112" — plausible numbers that
followed the deposit formula in catering.json. I skimmed and moved on.

verify_dataflow returned ok=False with unverified_facts=['£560','£112'].
The trace showed calculate_cost returned total_gbp=540, deposit=0. The
real total was £540 under the £300 deposit threshold. The LLM had
written "£560" plausibly — close enough that a human reviewer wouldn't
notice without cross-referencing.

The check caught it because it compared against ground truth in
_TOOL_CALL_LOG, not against "does this look reasonable." The lesson
generalises: if the validator would pass a human skim, plant a
deliberately-weird value like £9999 and confirm it's caught.

### Citation

- sessions/sess_de44a1b8eb12/workspace/flyer.md:12
- sessions/sess_de44a1b8eb12/logs/trace.jsonl:15

---

## Q3 — Removing one framework primitive

### Your answer

I'd keep session directories (Decision 1) as the last thing standing
and rebuild everything else if forced. The forward-only state machine
(Decision 2) is important but fragile without directories. Tickets
(Decision 3) I could rebuild as .jsonl files inside the session.
Atomic-rename IPC (Decision 5) is replaceable by directory polling.

Session directories are the irreplaceable piece. Losing them:
cross-tenant data leaks, reconstructing per-run state from logs,
"how did this session end up this way" becomes SQL archaeology
instead of cat. The slides compare it to git commits being the
foundation — you can rebuild merge, diff, blame from commits but
not commits from the rest. Session directories are commits.

### Citation

- sessions/sess_de44a1b8eb12/ — the directory itself
- sessions/sess_a382a2149fc1/logs/trace.jsonl
