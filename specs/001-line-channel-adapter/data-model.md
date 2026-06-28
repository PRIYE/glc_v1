# Data Model: LINE Channel Adapter

**Phase 1 output for `specs/001-line-channel-adapter`**
**Date**: 2026-06-26

---

## Entities

### 1. `LineWebhookEvent` (inbound wire format — parsed in `on_message`)

Represents a single event from the LINE Messaging API webhook POST body.

| Field | Type | Source (in raw dict) | Notes |
|-------|------|-----------------------|-------|
| `user_id` | `str` | `events[0].source.userId` | LINE user ID, format `U<hex>` |
| `text` | `str` | `events[0].message.text` | Message content (text type only) |
| `reply_token` | `str` | `events[0].replyToken` | One-shot, ~60s TTL |
| `message_id` | `str` | `events[0].message.id` | LINE message ID |
| `timestamp_ms` | `int` | `events[0].timestamp` | Milliseconds since Unix epoch |

**Validation rules**:
- `events` array must be non-empty and `events[0].type` must be `"message"`.
- `events[0].message.type` must be `"text"` for text processing (skip non-text silently).
- `reply_token` must be stored only when non-empty.

**State transitions**: Reply token starts as `available` after inbound parse → becomes `consumed` after the first `send` call → no longer in store.

---

### 2. `ReplyTokenStore` (in-memory state on the Adapter instance)

Tracks in-flight reply tokens per LINE user ID.

| Field | Type | Description |
|-------|------|-------------|
| `_tokens` | `dict[str, tuple[str, float]]` | Maps `user_id → (token, expiry_epoch)` |

**Operations**:
- `_stash(user_id, token, ttl_s=60.0)` — stores the token with expiry.
- `_consume(user_id) -> str | None` — pops and returns token if not expired; `None` otherwise.

**Lifecycle**: Created at adapter instantiation; lives for the adapter's lifetime. Tokens are short-lived (60 s TTL). No persistence across restarts.

---

### 3. `ChannelMessage` (outbound from `on_message`)

Canonical typed envelope. Defined in `glc/channels/envelope.py` — not redefined here.

| Field | Value for LINE |
|-------|----------------|
| `channel` | `"line"` (hardcoded) |
| `channel_user_id` | `events[0].source.userId` |
| `user_handle` | pairing record handle if available, else `channel_user_id` |
| `text` | `events[0].message.text` |
| `trust_level` | result of `classify("line", channel_user_id)` |
| `arrived_at` | `datetime.now(timezone.utc)` at parse time |
| `attachments` | `[]` (text-only for this assignment) |
| `metadata` | `{}` |

---

### 4. `ChannelReply` (inbound to `send`)

Canonical typed envelope. Defined in `glc/channels/envelope.py`.

| Field | Used by adapter |
|-------|-----------------|
| `channel_user_id` | Used as `to` in push payloads; also key into `ReplyTokenStore` |
| `text` | Placed in `messages[0].text` of outbound body |

---

### 5. LINE Outbound Payloads (produced by `send`)

**Reply payload** (when reply token available):
```json
{
  "replyToken": "<one-shot token>",
  "messages": [
    {"type": "text", "text": "<reply.text>"}
  ]
}
```

**Push payload** (fallback when no reply token):
```json
{
  "to": "<channel_user_id>",
  "messages": [
    {"type": "text", "text": "<reply.text>"}
  ]
}
```

---

## Relationships

```
LINE Webhook POST
       │
       ▼
 LineWebhookEvent (parsed)
       │
       ├─── reply_token ──► ReplyTokenStore._stash(user_id, token)
       │
       ├─── user_id ──► classify("line", user_id) ──► TrustLevel
       │
       └─── ──────────────► ChannelMessage (returned to agent runtime)

ChannelReply (from agent runtime)
       │
       └─► ReplyTokenStore._consume(user_id)
              │
         ┌───┴─────────────┐
         │ token found      │ no token
         ▼                  ▼
   Reply payload        Push payload
   {replyToken, msgs}   {to, msgs}
         │                  │
         └──────┬───────────┘
                ▼
         mock.send(payload) or real LINE API
```
