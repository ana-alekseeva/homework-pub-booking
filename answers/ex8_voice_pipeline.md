# Ex8 — Voice pipeline

## Your answer

Session `sess_d927def1d985` (text mode, 2026-05-24T21:29:40) records a
complete 6-turn conversation. The trace contains twelve events — alternating
`voice.utterance_in` and `voice.utterance_out` — all with `mode: "text"`:

```
turn 0 in:  "hey I wanna book a table"
turn 0 out: "How many's in your party?"
turn 1 in:  "10"
turn 1 out: "Too big, I'm afraid. Try The Royal Oak or Bennet's Bar,
             they can handle larger groups."
turn 2 in:  "ok, we have only 6 people"
turn 2 out: "Aye, we can do that. I'll pencil you in. What's the date
             and time?"
turn 3 in:  "25th June 7pm"
turn 3 out: "Aye, we can do that. I'll pencil you in for 25th June at
             7pm. What's the contact number?"
turn 4 in:  "984794839302"
turn 4 out: "Aye, got it. See you on the 25th."
turn 5 in:  "bye"
turn 5 out: "Cheerio."
```

The manager persona (Alasdair MacLeod) correctly enforced the 8-seat cap in
turn 1: party of 10 exceeds the cap, so it rejected and named two larger
alternatives (The Royal Oak, Bennet's Bar). When the user revised to 6 in
turn 2, the manager accepted immediately — 6 is within cap and deposit is £0
— and asked for date and time rather than completing, collecting the booking
details over the remaining turns. Every response was under 60 words.

The trace event format is identical whether the transport is text or voice:
`{"event_type": "voice.utterance_in", "actor": "user", "timestamp":
"2026-05-24T21:29:40.383016+00:00", "payload": {"text": "...", "turn": 0,
"mode": "text"}}`. The `mode` field distinguishes text from voice runs in
downstream analysis without branching the event schema.

Graceful degradation is implemented in `run_voice_mode` at the top of the
function: it checks `SPEECHMATICS_KEY` before importing `sounddevice` or
`speechmatics`. If the key is absent it falls back to `run_text_mode` with a
`sys.stderr` warning. The `OSError: PortAudio library not found` case —
raised by `sounddevice` when the system audio library is missing — is caught
by widening the handler to `except (ImportError, OSError)` so that missing
system libraries degrade gracefully rather than crashing.

`ManagerPersona` accumulates conversation history as a list of `ManagerTurn`
objects. Each call to `respond()` builds the full message list — system
prompt, then all prior turns interleaved as user/assistant messages, then
the new user message — before calling the LLM. This ensures the manager
remembers the party size and deposit from earlier in the conversation when
deciding whether to accept at the end.

## Citations

- `sessions/homework/ex8/sess_d927def1d985/logs/trace.jsonl` — 12 trace events, 6-turn negotiation (party 10 rejected → 6 accepted)
- `starter/voice_pipeline/voice_loop.py` — `run_text_mode` trace event emission, `run_voice_mode` graceful degradation
- `starter/voice_pipeline/manager_persona.py` — `ManagerPersona.respond()`, `_build_messages()`
