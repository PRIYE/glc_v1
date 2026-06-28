# Feature Specification: LINE Channel Adapter

**Feature Branch**: `001-line-channel-adapter`

**Created**: 2026-06-26

**Status**: Draft

**Input**: Implement the LINE Messaging API channel adapter for GLC v1 — `on_message` and `send` in `glc/channels/catalogue/line/adapter.py` with supporting Pydantic schemas in `schemas.py` so that all 7 tests in `tests/channels/test_line.py` pass against `LineMock`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Inbound Message Parsing & Trust Classification (Priority: P1)

A LINE user sends a text message to the bot. The gateway receives a webhook
POST body from LINE. The adapter parses the event, determines whether the
sender is the paired owner, a paired user, or untrusted, and returns a
fully-populated `ChannelMessage` envelope to the agent runtime.

**Why this priority**: This is the entry point to the entire adapter. Without
correct parsing and trust classification, no downstream logic can run.

**Independent Test**: Can be fully tested by calling `adapter.on_message(raw_webhook_event)` and
asserting the returned `ChannelMessage` has the correct `channel`, `channel_user_id`,
`trust_level`, `text`, and `arrived_at` fields.

**Acceptance Scenarios**:

1. **Given** an owner whose LINE user ID is registered as `owner_paired` in the pairing store, **When** a webhook event arrives with that user ID, **Then** `on_message` returns a `ChannelMessage` with `trust_level="owner_paired"` and the correct `text`.
2. **Given** an unrecognised sender (stranger), **When** a webhook event arrives with that user ID, **Then** `on_message` returns a `ChannelMessage` with `trust_level="untrusted"`.
3. **Given** a public-channel adapter (`is_public_channel=True`), **When** a stranger message arrives, **Then** `on_message` returns `None` or a message with `trust_level="untrusted"` (silently filtered).

---

### User Story 2 - Reply-Token-First Outbound (Priority: P1)

After receiving a message, the agent runtime calls `adapter.send(reply)`.
The adapter must use the LINE Reply API (quota-free) when an in-flight reply
token exists for that user, and fall back to the Push API (quota-counted)
only when no valid token is available.

**Why this priority**: Always using Push would exhaust the 500 msg/month free
quota. Correct token management is the key quota-optimisation behaviour.

**Independent Test**: Can be fully tested by (a) queueing an inbound message to
prime a reply token, (b) calling `send` once and asserting the payload contains
`replyToken`, then (c) calling `send` again and asserting the payload contains
`to` (push) and no `replyToken`.

**Acceptance Scenarios**:

1. **Given** an inbound message was processed (reply token primed), **When** `send` is called for the first outbound, **Then** the wire payload contains `{"replyToken": "<token>", "messages": [{"type": "text", "text": "..."}]}`.
2. **Given** no reply token is in flight (second consecutive reply), **When** `send` is called, **Then** the wire payload contains `{"to": "<user_id>", "messages": [{"type": "text", "text": "..."}]}` and does NOT include `replyToken`.
3. **Given** any outbound, **When** `send` is called, **Then** the `messages` array contains at least one object with `"type": "text"` and the correct `"text"` value.

---

### User Story 3 - Resilience & Error Propagation (Priority: P2)

The adapter handles network disruptions (forced disconnects) without raising
unhandled exceptions, and correctly surfaces LINE API rate-limit errors (HTTP 429)
back to the caller as a `{"status": 429}` dictionary instead of crashing.

**Why this priority**: Production reliability. Unhandled exceptions would crash
the channel control plane. Rate-limit propagation lets the caller decide on retry.

**Independent Test**: Can be fully tested independently by (a) calling `force_disconnect()`
on the mock then sending/receiving, and (b) setting `mock.rate_limited = True` then
calling `send` and asserting the returned dict has `status == 429`.

**Acceptance Scenarios**:

1. **Given** the mock is in forced-disconnect state, **When** `on_message` is called, **Then** the call completes without raising any exception.
2. **Given** the LINE API is rate-limiting (mock `rate_limited=True`), **When** `send` is called, **Then** the return value is a dict with `{"status": 429}`.

---

### Edge Cases

- What happens when the webhook `events` array is empty or contains no `message` type events?
- How does the adapter behave when a reply token has expired (TTL elapsed)?
- What if the `text` field in the webhook message is absent or `None`?
- What if `channel_user_id` in `ChannelReply` has no corresponding entry in the pairing store?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The adapter MUST parse the LINE webhook `events[0]` object and extract `source.userId`, `message.text`, and `replyToken`.
- **FR-002**: The adapter MUST resolve trust level for every inbound sender by consulting the pairing store via `glc.security.pairing.get_pairing_store().lookup("line", user_id)`.
- **FR-003**: The adapter MUST store the extracted `replyToken` in an in-memory TTL dictionary keyed by `user_id` so it can be consumed by a subsequent `send` call.
- **FR-004**: `send` MUST prefer the Reply endpoint (`{"replyToken": ..., "messages": [...]}`) when a valid, unconsumed reply token exists for the target user.
- **FR-005**: `send` MUST fall back to the Push endpoint (`{"to": ..., "messages": [...]}`) when no valid reply token is available.
- **FR-006**: Reply tokens MUST be consumed (removed from the store) when used, ensuring they are one-shot.
- **FR-007**: `on_message` MUST return `None` (or a message with `trust_level="untrusted"`) for strangers when the adapter is configured with `is_public_channel=True`.
- **FR-008**: A forced disconnect MUST be handled gracefully — `on_message` MUST NOT raise an unhandled exception when `mock.pop_disconnect()` returns `True`.
- **FR-009**: A rate-limit response from the send transport MUST be propagated to the caller as `{"status": 429}`.
- **FR-010**: The outbound `messages` array MUST contain at least one object with `"type": "text"` and the reply text.

### Key Entities

- **Webhook Event**: Represents a single LINE inbound event — contains `source.userId`, `message.text`, `replyToken`, and `timestamp`.
- **Reply Token Store**: An in-memory mapping of `user_id → (token, expiry_timestamp)`. Supports set, consume (pop), and TTL check.
- **ChannelMessage**: The typed envelope returned by `on_message` — fields: `channel`, `channel_user_id`, `user_handle`, `text`, `trust_level`, `arrived_at`.
- **ChannelReply**: The typed envelope consumed by `send` — fields: `channel`, `channel_user_id`, `text`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All 7 tests in `tests/channels/test_line.py` pass with zero failures or errors.
- **SC-002**: `pytest --cov=glc --cov-fail-under=80` continues to pass after the adapter is added.
- **SC-003**: `ruff check glc/channels/catalogue/line/` and `mypy glc/channels/catalogue/line/` both report zero issues.
- **SC-004**: The reply-vs-push selection is correct 100% of the time in the test suite — no false push calls when a reply token is available.
- **SC-005**: The adapter handles all edge-case scenarios (disconnect, rate-limit, public channel) without crashing the test process.

## Assumptions

- The LINE webhook body always has at least one event in the `events` array; the adapter processes only `events[0]`.
- The `arrived_at` timestamp is derived from `datetime.utcnow()` (or `datetime.now(timezone.utc)`) at parse time, not from the LINE `timestamp` field (which is milliseconds since epoch).
- `user_handle` in `ChannelMessage` defaults to the `channel_user_id` when no pairing record provides a handle.
- The mock (`LineMock`) is the sole transport in tests; production LINE API calls are out of scope for this assignment.
- No HMAC webhook signature verification is required for the test suite (no `LINE_CHANNEL_SECRET` is injected into the mock environment).
- The `is_public_channel` config key defaults to `False` when absent.
