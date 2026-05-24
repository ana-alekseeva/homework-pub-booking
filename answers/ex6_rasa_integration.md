# Ex6 — Rasa structured half

## Your answer

`RasaStructuredHalf.run()` in `starter/rasa_half/structured_half.py`
overrides the abstract `run()` method. The input flow is: loop half
produces raw booking data as a dict (keys include `venue_id`, `date`,
`time`, `party_size`, `deposit`) → `normalise_booking_payload()` in
`validator.py` canonicalises all five fields and raises `ValidationFailed`
if any is missing or unparseable → the normalised dict is JSON-encoded and
POSTed to Rasa's REST webhook at `/webhooks/rest/webhook` → the response
is parsed for `next_action: committed` or `next_action: escalate`.

The validator normalises all five graded fields. `canonicalise_venue_id`
converts `"Haymarket Tap"` → `"haymarket_tap"` by lower-casing and
replacing whitespace with underscores. `parse_currency_gbp` handles `"£500"`,
`"500 GBP"`, and bare `500`. `parse_time_24h` converts `"7:30pm"` → `"19:30"`
and `"half seven"` is handled by the 12-hour regex path. `_normalise_date`
handles ISO-8601 pass-through, relative words (`"today"` → `"2026-04-25"`,
`"tomorrow"` → `"2026-04-26"`), and natural-language formats like `"25th
April"`. `parse_party_size` extracts the leading integer from strings such
as `"6 people"`.

Three design choices worth recording: (1) `ValidationFailed` is a
`ValueError` subclass caught inside `run()` — the `StructuredHalf` contract
requires a `HalfResult` return value, never a raised exception.
(2) `urllib.error.URLError` and `http.client.RemoteDisconnected` are caught
and mapped to `HalfResult(success=False, next_action="escalate")` with error
code `SA_EXT_SERVICE_UNAVAILABLE`, so the bridge caller decides whether to
retry rather than crashing. (3) The Rasa `sender` field is set to
`homework-{sha1(venue_id+date+time)[:8]}` — a deterministic hash of the
booking key — so that Rasa's conversation tracker stays consistent across
retries within the same session without generating a new tracker per attempt.

For offline testing without a Rasa Pro license, `spawn_mock_rasa()` starts a
`ThreadingHTTPServer` on a free port that always returns
`[{"text": "booking confirmed", "custom": {"next_action": "complete"}}]`.
This lets the HTTP-contract layer be verified independently of the Rasa
container.

## Citations

- `starter/rasa_half/validator.py` — `normalise_booking_payload`, `canonicalise_venue_id`, `parse_currency_gbp`, `parse_time_24h`, `_normalise_date`, `parse_party_size`
- `starter/rasa_half/structured_half.py` — `RasaStructuredHalf.run()`, `spawn_mock_rasa()`
