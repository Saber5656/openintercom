# 13: WebSocket endpoint, connection registry, heartbeat

## Summary

`internal/ws`: the WSS transport layer — upgrade with origin check,
first-message auth, envelope decoding, per-connection limits, heartbeat,
single-session policy, and a registry other modules subscribe to.

## Context

DESIGN §6.8 fixes the wire contract. This issue delivers transport semantics
only; message *meaning* (calls, roster) is issues 14–16, which consume the
registry through interfaces defined here.

## Scope

- `internal/ws/` (endpoint.go, conn.go, registry.go, envelope.go)
- Route `GET /api/v1/ws` (HTTPS listener only)

## Detailed Requirements

1. Library: `github.com/coder/websocket` (ADR-004). Accept options: compression
   disabled (tiny messages), `InsecureSkipVerify: false` — implement the origin
   check ourselves for exact semantics: allowed origins = every
   `https://<san>` and `https://<san>:<https-port>` for each leaf SAN
   (recomputed on cert rotation via `ca.Manager.LeafInfo`); missing or
   non-matching `Origin` → HTTP 403 before upgrade.
2. Envelope codec per §6.8: strict JSON decode into
   `{V int, Type string, Seq *int64, Payload json.RawMessage}`; reject
   (`close 4400`): `V != 1`, unknown `Type` (list owned by this package as
   `KnownTypes`, extended by 14/16 registrations), frame >128 KiB
   (`SetReadLimit`), non-text frames.
3. Auth handshake: after upgrade start a 5 s timer; first message must be
   `auth {token}`; validate via the station-token service from issue 11's
   middleware internals (extract shared `auth.StationResolver` helper);
   failure or timeout → send `auth.err {code:"unauthorized"}`, close 4401.
   Success → build `auth.ok` payload from providers (interfaces below), then
   mark the connection authenticated.
4. Providers consumed (stub-able):

   ```go
   type SnapshotProvider interface {
     RosterFor(stationID string) json.RawMessage      // roster array (§6.8)
     ActiveCallFor(stationID string) json.RawMessage  // CallState or null
     SettingsSnapshot() json.RawMessage               // {ring_timeout_s}
   }
   type Handler interface {   // implemented by signal router (16) & roster (14)
     OnConnect(stationID string, c Sender)
     OnMessage(stationID string, env Envelope) // called only when authenticated
     OnDisconnect(stationID string)
   }
   type Sender interface { Send(type_ string, payload any) error; Close(code int) }
   ```
5. Single-session policy: registry map `stationID → conn`; a new authenticated
   connection for an already-connected station closes the old one with 4409
   (**newest wins**), then replaces it; `OnDisconnect` fires for the old conn
   before `OnConnect` of the new (ordering guaranteed, single registry mutex).
6. Heartbeat: respond `pong{t}` to `ping{t}` immediately; server-side read
   deadline 60 s rolling on any inbound frame; expiry → close 1001-equivalent
   (use 4400? — no: normal close with going-away semantics; document code) and
   `OnDisconnect`.
7. Per-connection inbound rate limit: token bucket 30 msg/s burst 60; on trip
   send `error {code:"rate_limited"}` once, drop excess for 1 s, and close
   4400 if abuse continues ≥3 consecutive windows.
8. Outbound: `Sender.Send` marshals the §6.8 envelope (no `seq`), per-conn
   buffered channel (cap 64); a full buffer (slow client) → close the
   connection (backpressure = disconnect; client will resync via `auth.ok`).
9. `last_seen_at` persistence: on any inbound message, throttled to once/min
   per station (reuse issue 11 throttle helper).
10. Write timeout 10 s per frame; graceful shutdown: registry close-all with
    1001 on server stop (hooks into issue 03 Runner).

## Acceptance Criteria

- [ ] Upgrade without/with-wrong Origin → 403 pre-upgrade; correct Origin
      (name and IP-SAN variants) succeeds.
- [ ] No auth within 5 s → `auth.err` + close 4401 (fake clock).
- [ ] Bad token → 4401; good token → `auth.ok` containing roster, settings,
      `active_call:null` from stub providers.
- [ ] Second login same station: old conn gets 4409, handlers see
      Disconnect(old)→Connect(new) in order.
- [ ] `v:2` envelope, unknown type, 129 KiB frame, binary frame → 4400 each.
- [ ] ping→pong echo `t`; silent 61 s → disconnect.
- [ ] Flood 100 msg/s → one `rate_limited` error then close within 3 s.
- [ ] Registry `Send` after close returns error, never panics (race test).

## Validation

`go test -race ./internal/ws/...` using httptest TLS server + real client
conns; deterministic clocks injected; 200-connection churn stress test under
`-race`.

## Dependencies

07, 08 (route mount, TLS), 11 (token resolver helper).

## Non-goals

Roster computation (14), call semantics/relay (16), client implementation
(20).

## Design References

DESIGN.md §6.8, §6.9 hooks, §9.2, §9.6, §9.7.
