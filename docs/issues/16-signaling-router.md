# 16: Signaling router: wire WS ↔ state machine ↔ relay

## Summary

`internal/signal/router.go`: the single-goroutine event loop that drives the
callsm SessionTable from WS messages, executes its effects (pushes, relays,
timers, events), and provides the live-state interfaces consumed by roster
(14), admin API (12), and WS auth resync (13).

## Context

DESIGN §6.10–§6.11 semantics become runtime behavior here. Concurrency
model: **one event loop goroutine** owns the SessionTable; everything enters
through a channel; timers post back into the same channel. This makes the
`-race` story trivial and matches callsm's single-driver contract.

## Scope

- Router (event loop, timer wheel, effect executor)
- `ws.Handler` implementation for call/webrtc message types
- `BusyProvider` (for 14), `LiveInfoProvider`+`SignalHooks` (for 12),
  `SnapshotProvider.ActiveCallFor` (for 13)
- Event-log emission for `call.*` and hook-driven types

## Detailed Requirements

1. Inbox: `chan routerMsg` (cap 256) carrying: inbound WS envelopes
   (`call.invite|accept|decline|cancel|hangup`, `call.media`,
   `webrtc.offer|answer|ice`), connect/disconnect notifications, revocation
   hooks, timer firings, and synchronous query requests (snapshot reads use
   a reply channel — keep reads out of lock trouble). Full inbox: drop the
   message and close the offending connection (backpressure consistent with
   issue 13 §8).
2. Message → callsm mapping: strict payload validation first (§9.6): session
   ids RFC 4122; `call.invite.callee_id` RFC 4122; `webrtc.*` raw payload
   size measured before handing `Relay(kind,size)` to callsm. On callsm
   error effects, translate to WS `error` messages with `ref` = envelope
   seq.
3. Effect execution:
   - `PushState` → `Sender.Send("call.state", view)` via ws registry; absent
     connection = no-op (callsm already accounts for presence).
   - `RelayTo` → forward the **original raw payload** (opaque, §6.11) as the
     same message type to the peer.
   - `StartTimer/CancelTimer` → timer wheel (`time.AfterFunc` posting a
     timer routerMsg; cancellation via stored `*time.Timer`); ring duration
     comes from `Settings.RingTimeout()` read at invite time.
   - `AppendEvent` → store Events DAO (`call.started`, `call.answered`,
     `call.missed`, `call.ended` with `duration_s` = now−AnsweredAt).
4. Interfaces provided:
   - `IsBusy(stationID)` — table lookup (serialized via query message or an
     atomic snapshot map maintained by the loop; choose atomic
     copy-on-transition map for lock-free reads, documented).
   - `LiveInfoProvider.Snapshot()` — merges ws-registry online set,
     busy map, and roster mic states (collaborate with 14: roster owns mic;
     provider composition happens in `serve` wiring — this issue exposes
     busy+online, 14 exposes mic; issue 12's provider is a composite —
     implement the composite here as `signal.CompositeLiveInfo`).
   - `ActiveCallFor(stationID)` — current SessionView JSON or null (13's
     auth resync; §6.8 auth.ok).
   - `SignalHooks.StationRevoked(id)` → inbox revocation (callsm `Revoked`,
     T13) **and** ws close 4401; `StationRenamed(id)` → roster invalidate.
5. Reconnect grace (T12): ws `OnDisconnect` → callsm `Disconnected`;
   `OnConnect` → `Reconnected`. A client resyncing via `auth.ok.active_call`
   that answers with `call.hangup` for a session it doesn't know follows the
   normal hangup path (§6.10 note).
6. Roster busy invalidation: after each transition batch, if any station's
   busy bit changed, call `roster.Invalidate()`.
7. Shutdown: stop accepting inbox writes, end all live sessions
   `error`-reason effects executed best-effort, then return (Runner).
8. Logging: every transition at `debug` (T-row, session id, states),
   call summaries at `info` (§6.14); never log SDP/candidate contents.

## Acceptance Criteria

- [ ] Full happy choreography against two fake `ws.Sender`s: invite →
      both get `ringing` views (correct roles/peers) → accept → `connecting`
      → offer/answer/ice relayed verbatim (byte-equal payloads) →
      `call.media` → `active` → hangup → `ended/hangup` + events
      `call.started/answered/ended` in store with sane duration.
- [ ] Timeout path with fake-timer harness: no accept within configured
      ring timeout → `ended/timeout` + `call.missed` event.
- [ ] Disconnect matrices: mid-ring caller/callee drop (immediate end);
      mid-call drop → grace → `peer_disconnected` after 15 s unless
      Reconnected.
- [ ] Revocation mid-call: survivor gets `ended/error`; revoked conn closed
      4401.
- [ ] Busy map / ActiveCallFor / Snapshot reflect transitions promptly
      (read-after-transition test).
- [ ] `-race` clean under a 50-goroutine message storm; goleak clean on
      shutdown.

## Validation

`go test -race ./internal/signal/...` with fake registry/senders, fake
timers where deterministic and real timers in one slow-path test; goleak.

## Dependencies

13, 14, 15.

## Non-goals

Browser-side WebRTC (23), scripted end-to-end over real sockets (17), admin
polling (12 already consumes the interface).

## Design References

DESIGN.md §6.8–§6.12, §10; ADR-003.
