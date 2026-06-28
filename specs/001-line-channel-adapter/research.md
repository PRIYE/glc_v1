# Research: LINE Channel Adapter

**Phase 0 output for `specs/001-line-channel-adapter`**
**Date**: 2026-06-26

---

## Decision 1: How to represent and consume reply tokens

**Decision**: Use a plain `dict[str, tuple[str, float]]` in-memory store on the `Adapter` instance, keyed by `user_id` with value `(token_str, expiry_epoch)`. Expose `_stash_token(user_id, token)` and `_consume_token(user_id) -> str | None` as private helpers.

**Rationale**: The adapter must manage its own `_token_store` independently of the mock, ensuring that the TTL logic is natively tested when `send` decides between a Reply and a Push payload. Relying on the mock's `consume_reply_token` would delegate core state management to the test mock, leaving the adapter's production logic untested.

**Alternatives considered**:
- `asyncio.Queue` per user — over-engineered for single-threaded test scenario.
- Passing `replyToken` through `ChannelMessage.metadata` — violates Principle I (would put wire-format data inside the typed envelope reaching the agent).

---

## Decision 2: How `on_message` handles forced disconnect

**Decision**: At the start of `on_message`, check `mock.pop_disconnect()` if mock is present. If True, ignore it and proceed with parsing the webhook normally.

**Rationale**: Test `test_disconnect_is_handled` catches all exceptions and calls `pytest.fail` if any is raised. LINE uses Webhooks (HTTP POST requests), not persistent WebSockets, so a "disconnect" doesn't actually mean anything in the context of receiving a webhook. Proceeding normally is closer to how a real webhook endpoint behaves.

**Alternatives considered**:
- Raising a custom `DisconnectError` — test explicitly rejects this.
- Returning `None` — would require a `ChannelMessage | None` return type annotation change.

---

## Decision 3: Wire payload shape for Reply vs Push

**Decision**:
- Reply: `{"replyToken": token, "messages": [{"type": "text", "text": reply.text}]}`
- Push: `{"to": user_id, "messages": [{"type": "text", "text": reply.text}]}`

**Rationale**: Test `test_send_emits_valid_wire_payload` asserts `"messages" in body`, `body["messages"]` is a list, `first["type"] == "text"`, and `first["text"] == "hi back"`. Test `test_channel_specific_behaviour_reply_token_then_push` asserts `"replyToken" in body1` for first call, then `"to" in body2` and `body2["to"] == OWNER_ID` and `"replyToken" not in body2` for second call.

**Alternatives considered**:
- Bare `{"text": "..."}` — rejected; test explicitly checks for `messages` array.

---

## Decision 4: Trust level classification API

**Decision**: Use `glc.security.trust_level.classify(channel, channel_user_id)` — a single call returning `"owner_paired" | "user_paired" | "untrusted"`.

**Rationale**: The function already consults the pairing store singleton. The `pair_owner` fixture in the test calls `store.force_pair_owner("line", OWNER_ID)` which registers the user. `classify("line", OWNER_ID)` then returns `"owner_paired"`. No direct pairing-store access is needed in the adapter.

**Alternatives considered**:
- Calling `get_pairing_store().lookup()` directly — works but duplicates logic already in `classify()`.

---

## Decision 5: `user_handle` field in `ChannelMessage`

**Decision**: Use the `user_handle` from the pairing record if available; fall back to the `channel_user_id` string.

**Rationale**: `ChannelMessage.user_handle` is a required non-optional field. The pairing record's `user_handle` field is populated by `force_pair_owner("line", OWNER_ID, user_handle="owner")` in the test fixture. For untrusted senders with no pairing record, fall back to `channel_user_id`.

**Alternatives considered**:
- Always using `channel_user_id` — fails if user_handle were asserted (it is not currently in tests, but keeping it correct avoids future regressions).

---

## Decision 6: `arrived_at` timestamp source

**Decision**: Use `datetime.now(timezone.utc)` at parse time inside `on_message`.

**Rationale**: The test asserts `isinstance(msg.arrived_at, datetime)` only. The LINE webhook `timestamp` field is milliseconds since epoch — converting it would also work but adds complexity with no test benefit. Using `datetime.now(timezone.utc)` is the pattern recommended in `docs/ADAPTER_GUIDE.md`.

**Alternatives considered**:
- `datetime.utcnow()` — deprecated in Python 3.12; prefer timezone-aware.
- Parsing from webhook `timestamp` field — valid but unnecessary for test compliance.

---

## Summary: No NEEDS CLARIFICATION items remain

All six decisions above are fully resolved from the test suite contract and the existing codebase. Implementation can proceed directly.
