# Ex6 — Rasa structured half

## Your answer

The `RasaStructuredHalf` subclass overrides `run()` to POST a booking
intent to Rasa's REST webhook and interpret the response. Input flows:
the loop half produces raw booking data, `StructuredHalf` calls
`normalise_booking_payload` (in validator.py) to produce a Rasa-shaped
message with canonical types, sends it to Rasa via the REST webhook, and
parses the response for `{committed: true}` or a rejection.

I ran the full three-process flow (Terminal 1: `make rasa-actions` on
:5055; Terminal 2: `make rasa-serve` on :5005; Terminal 3: `make ex6-real`).
Rasa trained a CALM model and the booking committed end-to-end:
`{committed: True, venue_id: haymarket_tap, party_size: 6,
deposit_gbp: 200, booking_reference: BK-7D401E9E}` (session
sess_650e7ded7074). The £200 deposit was under the £300 auto-approve
ceiling, so the manager's structured half accepted it; my custom
`action_validate_booking` ran inside Rasa to enforce that rule.

For offline mode a stdlib http.server thread mimics the Rasa webhook.
It always confirms, which is enough for the HTTP-contract unit tests;
rejection is exercised in Ex7 where the loop half's arguments drive the
decision.

Three design choices worth noting: (1) `normalise_booking_payload`
raises `ValidationFailed` which `run()` catches, rather than letting it
propagate — the `StructuredHalf` contract demands a `HalfResult`. (2)
Network errors return `success=False` with `SA_EXT_SERVICE_UNAVAILABLE`
so the caller decides whether to retry. (3) The stable `sender_id` is a
hash of (venue+date+time), so the Rasa tracker stays consistent across
retries within one session.

## Citations

- starter/rasa_half/validator.py — normalise_booking_payload + helpers
- starter/rasa_half/structured_half.py — RasaStructuredHalf.run + mock server
- ~/.local/share/sovereign-agent/examples/ex6-rasa-half/sess_650e7ded7074 — committed run (BK-7D401E9E)