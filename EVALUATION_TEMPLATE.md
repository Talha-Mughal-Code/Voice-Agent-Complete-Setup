# Evaluation Results

Architecture: Retell (streaming STT, LLM, streaming TTS, barge-in, telephony) →
`POST /retell/*` on the mock reservation API via `retell_adapter.py`.

| Test | Pass/Fail | Final outcome | Tool calls | Duplicate/wrong write? | End-of-speech to first audio | API latency | Notes |
|---|---|---|---|---|---:|---:|---|
| T1 | Pass | `LUMA-9D59` created, exactly 1 record, party 4 at 18:00 | availability, create, search | no | 820–1150 ms | 13 ms | Phone normalised to E.164 by the adapter; hyphenated input would otherwise match nothing |
| T2 | Pass | Unavailable at 18:30; offered 17:30 / 18:00 / 19:30; booked 19:30 | availability, create | no | 820–1150 ms | 12 ms | Alternatives taken verbatim from the API response; none invented |
| T3 | Pass | 1 record, final party_size 4 | availability ×2, create | no | 820–1150 ms | 11 ms | Availability re-checked after the correction; superseded party size never written |
| T4 | Pass | `res_existing_4821` → 19:30, party 4 | search, modify | no | 820–1150 ms | 12 ms | Adapter resolves `reservation_id` from the confirmation code |
| T5 | Pass | Cancelled once; repeat returns 409 `ALREADY_CANCELLED` | search, cancel ×2 | no | 820–1150 ms | 11 ms | Bare "4821" accepted; repeat cancel does not double-credit capacity |
| T6 | Pass | 503 on first call, 200 on one retry | availability ×2 | no | 820–1150 ms | 14 ms | Retry is agent-driven — Retell does not retry custom functions |
| T7 | Pass | `LUMA-DF8F` returned on all 3 creates, 1 record | create ×3, search | no | 820–1150 ms | 12 ms | `Idempotency-Key` derived server-side from the booking fields; agent sends no header |

## Aggregate

- **Task success rate: 7/7 (100%)**
- **Tool-call accuracy: 100%** — every scenario used the expected endpoints with valid arguments; no invented availability, no invented confirmation codes
- **Duplicate-write rate: 0%** — T7 sent the same booking three times (twice identical, once with notes added) and produced one record with one confirmation code; capacity debited once
- **End-of-speech to first audio: 820–1150 ms observed** across live calls (~985 ms mid-range). This is the full voice loop: ASR finalisation + LLM + tool round trip + TTS first byte.
- **API latency: 12 ms average, 17 ms worst case** over 22 calls, measured at the endpoint. So the tool layer is ~1–2% of perceived latency; the remainder is ASR/LLM/TTS.
- **p50 / p95 API latency: 12 ms / 16 ms**

Method: voice latency from live Retell calls. Endpoint outcomes, side-effect
assertions and API latency from `tests/run_adapter_tests.py`, which drives the
same `/retell/*` endpoints the agent calls and verifies record counts and
capacity after each scenario (`tests/adapter_results.json`).

## Known limitations

- Availability exists only for 2026-08-14, 08-15 and 08-16; any other date returns 422. The agent is prompted to offer only those three evenings, and `current_date` is pinned to 2026-08-13 so relative dates resolve inside that range.
- All state is in-process Python dicts (`app.py:14-18`). Restarting the API wipes every booking and restores capacity; two replicas would each hold separate reservations *and* separate idempotency tables, so duplicate prevention would break horizontally.
- Retry, the confirm-before-write gate, and 12-hour → 24-hour time conversion are prompt-enforced rather than code-enforced on this direct-connect path. `"6:00 PM"` sent literally returns 422 `INVALID_SLOT`.
- Retell does not retry custom functions, so the single retry after a 503 depends on the model following instruction.
- The API is reached over an ngrok tunnel with no authentication of its own; the hostname changes on restart and is baked into the tool URLs.
- Forcing two consecutive tool failures is not possible with this mock (the 503 fires once per reset), so failure-driven handoff was exercised via the party-size-over-8 policy path instead.
