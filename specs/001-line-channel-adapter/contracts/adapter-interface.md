# Adapter Interface Contract: LINE Channel Adapter

**Phase 1 output — contracts/**
**Date**: 2026-06-26

The adapter exposes exactly two public methods as required by `glc.channels.base.ChannelAdapter`.

---

## `Adapter.on_message(raw: Any) -> ChannelMessage | None`

### Input

`raw` is a `dict` matching the LINE Messaging API webhook POST body:

```json
{
  "destination": "<bot_user_id>",
  "events": [
    {
      "type": "message",
      "mode": "active",
      "timestamp": 1700000000000,
      "source": {
        "type": "user",
        "userId": "<LINE_user_id>"
      },
      "replyToken": "<one-shot-token>",
      "message": {
        "id": "<message_id>",
        "type": "text",
        "text": "<message text>"
      }
    }
  ]
}
```

The mock produces this shape via `LineMock.queue_owner_message()` and `LineMock.queue_stranger_message()`.

### Output

Returns a `ChannelMessage` (from `glc.channels.envelope`) on success.

Returns `None` when the adapter is configured as a public channel and the sender is not on the allowlist (or is untrusted with `mention_only_in_public` active).

Raises **nothing** — including on forced disconnect.

### Side-effects

- Stores the reply token in the internal `ReplyTokenStore` keyed by `source.userId`.

### Guarantees

| Guarantee | Condition |
|-----------|-----------|
| `msg.channel == "line"` | Always |
| `msg.channel_user_id == events[0].source.userId` | Always |
| `msg.trust_level == "owner_paired"` | When `channel + user_id` is paired as owner in the pairing store |
| `msg.trust_level == "untrusted"` | When `channel + user_id` is not in the pairing store |
| `isinstance(msg.arrived_at, datetime)` | Always |
| No exception raised | Always, including on `mock.pop_disconnect() == True` |

---

## `Adapter.send(reply: ChannelReply) -> Any`

### Input

`reply` is a `ChannelReply` (from `glc.channels.envelope`):

```python
ChannelReply(channel="line", channel_user_id="<LINE_user_id>", text="<text>")
```

### Output

Returns the transport response dict.

- On success (mock not rate-limited): `{"sentMessages": [{"id": "sent-N"}]}`
- On rate limit: `{"status": 429, "message": "Too Many Requests"}`

### Wire payload dispatched

**When a valid reply token is in-flight for `reply.channel_user_id`**:
```json
{
  "replyToken": "<consumed-token>",
  "messages": [{"type": "text", "text": "<reply.text>"}]
}
```

**When no valid reply token is available**:
```json
{
  "to": "<reply.channel_user_id>",
  "messages": [{"type": "text", "text": "<reply.text>"}]
}
```

### Guarantees

| Guarantee | Condition |
|-----------|-----------|
| `"messages" in payload` | Always |
| `payload["messages"][0]["type"] == "text"` | Always |
| `payload["messages"][0]["text"] == reply.text` | Always |
| `"replyToken" in payload` | Only when a non-expired reply token was available |
| `"to" in payload` | Only when no reply token was available (push path) |
| `"replyToken" not in payload` | When using push path |
| Return value `{"status": 429}` | When `mock.rate_limited == True` |

---

## Configuration keys (passed via `config` dict)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mock` | `LineMock \| None` | `None` | If present, all outbound calls are dispatched through `mock.send(payload)` |
| `is_public_channel` | `bool` | `False` | When `True`, applies public-channel allowlist filtering |
