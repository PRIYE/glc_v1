# Implementation Plan: LINE Channel Adapter

**Branch**: `001-line-channel-adapter` | **Date**: 2026-06-26 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-line-channel-adapter/spec.md`

---

## Summary

Implement `on_message` and `send` in `glc/channels/catalogue/line/adapter.py` (and supporting Pydantic schemas in `schemas.py`) so that all 7 tests in `tests/channels/test_line.py` pass against `LineMock`. The key architectural requirement is a TTL-based reply-token store: the adapter must consume reply tokens (quota-free) on first outbound and fall back to push (quota-counted) when no token is available.

---

## Technical Context

**Language/Version**: Python 3.11

**Primary Dependencies**:
- `pydantic >= 2.6` — for `schemas.py` Pydantic models
- `glc.channels.base.ChannelAdapter` — ABC to subclass
- `glc.channels.envelope.ChannelMessage`, `ChannelReply` — typed envelopes
- `glc.security.trust_level.classify` — trust classification
- `glc.security.pairing.get_pairing_store` — pairing store lookup (via `classify`)
- `datetime` (stdlib) — `datetime.now(timezone.utc)` for `arrived_at`

**Storage**: In-memory `dict` on adapter instance for reply token TTL store. No persistent storage.

**Testing**: `pytest` with `pytest-asyncio` (async test mode). Mock: `tests/channels/mocks/line_mock.py`.

**Target Platform**: Python package — no network calls in tests (mock-driven).

**Project Type**: Channel adapter plugin within GLC v1 FastAPI gateway.

**Performance Goals**: N/A for this assignment (mock-driven tests).

**Constraints**: Must not modify `tests/` files; must not stray outside `glc/channels/catalogue/line/`.

**Scale/Scope**: 2 files, ~80–120 lines of implementation code total.

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Typed Envelopes | ✅ PASS | `on_message` returns `ChannelMessage`; `send` consumes `ChannelReply`. No raw LINE dicts reach the agent. |
| II. External Policy Engine | ✅ PASS | Adapter does not touch `glc/policy/`. Policy evaluation happens upstream of adapters. |
| III. Trust-Level Classification | ✅ PASS | `classify("line", user_id)` is called on every inbound message before constructing `ChannelMessage`. |
| IV. Test-First / Mock-Driven | ✅ PASS | Test file and mock file are fixed. Adapter is written to satisfy them. |
| V. Boundary-Scoped Changes | ✅ PASS | Only `glc/channels/catalogue/line/adapter.py` and `schemas.py` are written. |

**No gate violations. Proceeding.**

---

## Project Structure

### Documentation (this feature)

```text
specs/001-line-channel-adapter/
├── plan.md              ← This file
├── spec.md              ← Feature specification
├── research.md          ← Phase 0: all decisions resolved
├── data-model.md        ← Phase 1: entities and data flow
├── quickstart.md        ← Phase 1: validation scenarios
├── contracts/
│   └── adapter-interface.md  ← Phase 1: method contracts
├── checklists/
│   └── requirements.md  ← Spec quality checklist (all pass)
└── tasks.md             ← Phase 2 output (created by /speckit-tasks)
```

### Source Code (repository root)

```text
glc/channels/catalogue/line/
├── __init__.py          ← exists (no changes needed)
├── README.md            ← exists (no changes needed)
├── adapter.py           ← IMPLEMENT: on_message + send + ReplyTokenStore
└── schemas.py           ← IMPLEMENT: LineWebhookSource, LineWebhookMessage, LineWebhookEvent
```

**Structure Decision**: Single-file-per-concern within the pre-existing slot directory. No new directories needed.

---

## Implementation Design

### `schemas.py`

Three Pydantic models to parse the LINE webhook body safely:

```
LineWebhookSource:   userId: str
LineWebhookMessage:  id: str, type: str, text: str | None
LineWebhookEvent:    type: str, source: LineWebhookSource, message: LineWebhookMessage,
                     replyToken: str, timestamp: int
LineWebhookBody:     destination: str, events: list[LineWebhookEvent]
```

All fields mapped directly to the `_webhook()` helper in `line_mock.py`.

### `adapter.py` — `Adapter` class

**Class-level state**:
- `_token_store: dict[str, tuple[str, float]]` — reply token TTL store initialised in `__init__`.

**`_stash_token(user_id, token, ttl_s=60.0)`** — stores `(token, time.time() + ttl_s)`.

**`_consume_token(user_id) -> str | None`**:
- Pops from `_token_store`.
- Returns `None` if not found or expired.
- Returns token string if valid.

**`on_message(raw)` flow**:
1. Get mock from `self.config.get("mock")`.
2. If mock and `mock.pop_disconnect()` → ignore and proceed.
3. Parse `raw` with `LineWebhookBody(**raw)`.
4. Extract `event = body.events[0]`.
5. `user_id = event.source.userId`.
6. Stash reply token: `self._stash_token(user_id, event.replyToken)`.
7. Determine trust: `trust = classify("line", user_id)`.
8. (Removed: The gateway router enforces allowlists. The adapter simply returns the `ChannelMessage` with `trust_level="untrusted"`.)
9. Get `user_handle` from pairing store record or fall back to `user_id`.
10. Return `ChannelMessage(channel="line", channel_user_id=user_id, user_handle=handle, text=event.message.text, trust_level=trust, arrived_at=datetime.now(timezone.utc))`.

**`send(reply)` flow**:
1. Get mock from `self.config.get("mock")`.
2. `token = self._consume_token(reply.channel_user_id)`
3. Build payload:
   - If token: `{"replyToken": token, "messages": [{"type": "text", "text": reply.text}]}`
   - Else: `{"to": reply.channel_user_id, "messages": [{"type": "text", "text": reply.text}]}`
4. If mock: `return await mock.send(payload)`.
5. Else: dispatch to real LINE API (out of scope for test suite).

---

## Complexity Tracking

No constitution violations. No complexity justification required.
