# KFIRE WebSocket Protocol — Presence Events

Real-time presence is delivered over a single WebSocket endpoint:

```
wss://<host>/ws
```

This document is normative for both [kfire-client](https://github.com/knightsofeternity/kfire-client)
(desktop client) and [kfire-server](https://github.com/knightsofeternity/kfire-server),
as well as the admin web UI (browser consumer).

## Envelope

Every message — in both directions — is a JSON object with this envelope:

```json
{
  "type": "game_started",
  "ts": "2026-06-06T18:42:13Z",
  "payload": { }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Event name (see below) |
| `ts` | string (RFC 3339, UTC) | When the sender emitted the event |
| `payload` | object | Event-specific data; `{}` when an event carries none |

Unknown `type` values MUST be ignored by receivers (forward compatibility).

## Connection lifecycle

1. Client opens the WebSocket (no token in the URL — tokens in URLs leak into logs).
2. Client MUST send `hello` as its **first** message, within **10 seconds**.
3. Server replies `hello_ack` (or `error` + close on failure).
4. Client sends presence events as they happen, plus periodic `heartbeat`.
5. Server broadcasts `presence_update` to all authenticated connections of the org.

The access token is a 15-minute JWT. When it expires mid-connection the server
keeps the connection open — the connection was authenticated at `hello` time.
On reconnect the client MUST present a fresh token (refresh via REST first).

### Heartbeat & liveness

- Client sends `heartbeat` every **30 seconds**.
- Server marks a client connection dead after **90 seconds** without any message,
  closes it, and sets the user `offline` (ending any open client-sourced session).

### Reconnection

Clients MUST reconnect with exponential backoff + jitter:
`min(2^attempt × 1s + rand(0–1s), 60s)`. After reconnecting, the client MUST
re-send `game_started` for any game still running locally — the server
deduplicates against open sessions (same user + game ⇒ no new session).

---

## Client → Server events

### `hello`

First message on a connection. Authenticates it.

```json
{
  "type": "hello",
  "ts": "2026-06-06T18:42:13Z",
  "payload": {
    "protocol_version": 1,
    "access_token": "<jwt>",
    "device_id": "0d1f7e9a-...",
    "client": "kfire-client/0.1.0 (linux)"
  }
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `protocol_version` | yes | Currently `1`. Server rejects unsupported versions. |
| `access_token` | yes | Valid (non-expired) JWT access token |
| `device_id` | yes | Same device UUID used at login |
| `client` | no | Free-form client identification, for diagnostics |

### `game_started`

The local process scanner detected a known game starting.

```json
{
  "type": "game_started",
  "ts": "2026-06-06T18:45:02Z",
  "payload": {
    "game_slug": "counter-strike-2",
    "started_at": "2026-06-06T18:45:00Z"
  }
}
```

`started_at` is the detection time on the client clock; the server stores its own
receive time as authoritative and keeps `started_at` for diagnostics only.

### `game_stopped`

The previously reported game's process is gone.

```json
{
  "type": "game_stopped",
  "ts": "2026-06-06T20:03:44Z",
  "payload": {
    "game_slug": "counter-strike-2"
  }
}
```

### `heartbeat`

Keep-alive, every 30 s. Empty payload.

```json
{ "type": "heartbeat", "ts": "2026-06-06T18:45:30Z", "payload": {} }
```

---

## Server → Client events

### `hello_ack`

Authentication succeeded; connection is live.

```json
{
  "type": "hello_ack",
  "ts": "2026-06-06T18:42:13Z",
  "payload": {
    "protocol_version": 1,
    "heartbeat_interval_seconds": 30,
    "session_resumed": false
  }
}
```

`session_resumed` is `true` when the server still had an open game session for
this user (e.g. after a brief disconnect).

### `presence_update`

Broadcast to every connection of the org whenever any member's presence changes.
The payload mirrors `PresenceEntry` from `openapi.yaml`.

```json
{
  "type": "presence_update",
  "ts": "2026-06-06T18:45:02Z",
  "payload": {
    "user_id": "7b1c...",
    "username": "ouranos",
    "status": "in_game",
    "game": {
      "id": "3f2a...",
      "name": "Counter-Strike 2",
      "slug": "counter-strike-2",
      "icon_url": "https://..."
    },
    "since": "2026-06-06T18:45:02Z"
  }
}
```

`status` ∈ `offline | online | in_game`. `game` is `null` unless `in_game`.

### `error`

Sent before the server closes a connection on a protocol violation, or as a
non-fatal notice for a rejected event.

```json
{
  "type": "error",
  "ts": "2026-06-06T18:42:14Z",
  "payload": {
    "code": "auth_failed",
    "message": "access token expired",
    "fatal": true
  }
}
```

Defined codes:

| Code | Fatal | Meaning |
|------|-------|---------|
| `auth_failed` | yes | `hello` token invalid/expired, or no `hello` within 10 s |
| `unsupported_protocol` | yes | `protocol_version` not supported |
| `unknown_game` | no | `game_slug` not in the games database; event dropped |
| `rate_limited` | no | Client is sending events too fast; event dropped |

When `fatal` is `true` the server closes the connection immediately after.

---

## WebSocket close codes

| Code | Meaning |
|------|---------|
| 1000 | Normal closure (client quit) |
| 4001 | Authentication failed |
| 4002 | Unsupported protocol version |
| 4008 | Heartbeat timeout |
