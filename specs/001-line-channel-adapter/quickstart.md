# Quickstart & Validation Guide: LINE Channel Adapter

**Phase 1 output for `specs/001-line-channel-adapter`**
**Date**: 2026-06-26

This guide documents how to validate the LINE adapter implementation end-to-end.

---

## Prerequisites

1. Python 3.11+ with `uv` installed.
2. Dependencies synced: `uv sync` from the repo root.
3. No real LINE API credentials are required — the test suite uses a mock.

---

## Local Validation

### Run the full LINE test suite

```sh
uv run pytest tests/channels/test_line.py -v
```

**Expected outcome**: All 7 tests pass.

```
PASSED tests/channels/test_line.py::test_on_message_owner_returns_valid_envelope
PASSED tests/channels/test_line.py::test_on_message_stranger_is_untrusted
PASSED tests/channels/test_line.py::test_send_emits_valid_wire_payload
PASSED tests/channels/test_line.py::test_disconnect_is_handled
PASSED tests/channels/test_line.py::test_rate_limit_propagates_429
PASSED tests/channels/test_line.py::test_allowlist_silently_drops_stranger_in_public
PASSED tests/channels/test_line.py::test_channel_specific_behaviour_reply_token_then_push
```

### Verify ruff and mypy

```sh
uv run ruff check glc/channels/catalogue/line/
uv run ruff format --check glc/channels/catalogue/line/
uv run mypy glc/channels/catalogue/line/
```

**Expected outcome**: Zero errors/warnings from each command.

### Verify coverage gate is still met

```sh
uv run pytest tests/ --cov=glc --cov-fail-under=80 -m "not requires_live_api" -q
```

**Expected outcome**: Coverage remains at or above 80%.

---

## Scenario Walk-throughs

### Scenario A: Owner sends a message (reply token path)

1. Pair an owner: the test fixture calls `store.force_pair_owner("line", "Uowner", user_handle="owner")`.
2. Queue an inbound: `ev = mock.queue_owner_message("hello")` — this primes a reply token on the mock.
3. Parse: `msg = await adapter.on_message(ev)` — verify `msg.trust_level == "owner_paired"` and `msg.text == "hello"`.
4. Reply (first): `await adapter.send(ChannelReply(channel="line", channel_user_id="Uowner", text="hi"))` — verify `mock.send_log[-1]` contains `"replyToken"` and NOT `"to"`.

See `contracts/adapter-interface.md` for the exact expected payload shapes.

### Scenario B: Second consecutive reply (push fallback)

Continuing from Scenario A:
5. Reply (second): `await adapter.send(ChannelReply(channel="line", channel_user_id="Uowner", text="second"))` — verify `mock.send_log[-1]` contains `"to": "Uowner"` and does NOT contain `"replyToken"`.

### Scenario C: Stranger message (untrusted)

1. No pairing needed.
2. Queue: `ev = mock.queue_stranger_message("hi")`.
3. Parse: `msg = await adapter.on_message(ev)` — verify `msg.trust_level == "untrusted"`.

### Scenario D: Forced disconnect (resilience)

1. `mock.force_disconnect()`.
2. `await adapter.on_message(mock.queue_owner_message("x"))` — must complete without raising.

### Scenario E: Rate limit (error propagation)

1. `mock.rate_limited = True`.
2. `result = await adapter.send(ChannelReply(channel="line", channel_user_id="Uowner", text="x"))`.
3. Verify `result == {"status": 429, ...}`.

---

## Files Modified by Implementation

All changes are scoped to the owned paths listed in `GROUPS.md` for Group LINE:

```
glc/channels/catalogue/line/adapter.py   ← primary implementation
glc/channels/catalogue/line/schemas.py   ← Pydantic types for webhook parsing
```

Do **not** modify:
- `tests/channels/test_line.py`
- `tests/channels/mocks/line_mock.py`
- Any file outside `glc/channels/catalogue/line/`

---

## CI Checks (automated on PR)

The `adapter-pr.yml` workflow runs three jobs automatically:

1. **boundary** — ensures diff stays within `glc/channels/catalogue/line/`.
2. **test-changed-slot** — runs `pytest tests/channels/test_line.py` + ruff + mypy.
3. **scorecard** — comments a rubric scorecard on the PR.

For details on the rubric, see `docs/ADAPTER_GUIDE.md` §10.
