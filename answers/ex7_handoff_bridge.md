# Ex7 — Handoff bridge

## Your answer

The handoff bridge runs the loop and structured halves in alternating
rounds over a shared session, passing control via IPC files until one
half reports completion or max_rounds is hit. In my run, the bridge
completed in 2 rounds with outcome `completed` ("structured confirmed
in round 2").

Round 1: `HandoffBridge.run` invoked the loop half on the task ("book
for party of 12 in Haymarket"). The loop half produced booking data and
emitted a forward handoff to the structured half, written to
`ipc/output/` and archived under `logs/handoffs/`.

Round 2: the bridge fed that payload to the structured half via
`build_reverse_task`, which validated the booking against the venue's
policy (deposit cap, party size) and returned a confirmation. The bridge
saw the structured half's success, set the outcome to `completed`, and
stopped — short of max_rounds, so no MAX_ROUNDS_EXCEEDED escalation.

Two things the bridge enforces that matter: it caps rounds so a
ping-pong between halves can't loop forever, and the Ex7 integrity check
(`starter/handoff_bridge/integrity.py`) verifies the bridge actually did
work — an `outcome=completed` with zero rounds run would fail integrity,
guarding against a fake "complete" on turn 0.

## Citations

- starter/handoff_bridge/bridge.py — HandoffBridge.run + round loop
- starter/handoff_bridge/integrity.py — outcome/round-count verification
