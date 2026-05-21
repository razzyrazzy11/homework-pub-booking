# Ex8 — Voice pipeline

## Your answer

I ran the pipeline in voice mode (`make ex8-voice`, session
sess_30fa6d6a9586). Each turn ran the full audio loop: my spoken input
was captured via the mic, transcribed by Speechmatics (STT) into a
`voice.utterance_in` event; the manager agent (Alasdair, the pub-manager
persona) generated a reply; and that reply was synthesised by Rime (TTS),
played back through ffmpeg, and recorded as a `voice.utterance_out` event.

The conversation exercised both acceptance and rejection. I booked a
party of 6 for the 25th at 7:30pm and gave a contact number; Alasdair
confirmed and held the slot. I then asked for a £400 deposit, which
exceeds the £300 authorisation cap in the manager's system prompt. He
refused in character — "Too much, head office needs to sign off on that"
— rather than accepting. When I dropped to £200 (under the cap), the
booking completed ("Aye, that's fine. £200 deposit, see you on the 25th").

The run also demonstrated graceful degradation of the external services.
In an earlier attempt the Rime API key was invalid; the pipeline logged
`Rime 401: invalid api key (continuing)` and proceeded without spoken
output rather than crashing — TTS failure is non-fatal and the
conversation still completed end-to-end. Once the key and ffmpeg were
fixed, full audio playback worked. Party size (6) was under the 8-person
cap, so only the deposit constraint was triggered this run.

## Citations

- starter/voice_pipeline/voice_loop.py — run_voice_mode, STT/TTS, utterance events
- starter/voice_pipeline/manager_persona.py — £300 cap, party-size-8 limit
- sessions/sess_30fa6d6a9586/logs/trace.jsonl — voice run; utterance_in/utterance_out events
