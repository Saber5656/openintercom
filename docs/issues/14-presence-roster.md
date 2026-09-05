# 14: Presence and roster broadcast

## Summary

`internal/signal/roster.go`: track who is online/busy/mic-state and push
coalesced full-snapshot `roster.update` messages to every connected station.

## Context

DESIGN §6.9. The roster is the home screen's data source and the admin
station list's live component. Busy state comes from the call table (issue
15/16) through a provider interface; this issue ships with a stub.

## Scope

- Roster tracker + broadcaster registered as a `ws.Handler`
- `station.status` message handling (mic ok/blocked)
- `RosterFor` / snapshot provider implementations for issues 12/13

## Detailed Requirements

1. State per station id: `online` (from ws OnConnect/OnDisconnect),
   `mic` ("unknown" until a `station.status` arrives on each new connection),
   `busy` via `type BusyProvider interface { IsBusy(stationID string) bool }`
   (stub: always false; issue 16 provides the real one and calls
   `roster.Invalidate()` on session transitions).
2. Roster entry per §6.8: `{id, name, icon, online, busy, mic}` — name/icon
   read through a small read-through cache over the store, invalidated by
   `SignalHooks.StationRenamed/StationRevoked` (wire those from issue 12's
   hook interface — this issue implements the roster side of the hooks).
3. Broadcast policy: any change (connect, disconnect, mic change, rename,
   revoke, busy invalidation) marks dirty; a coalescing loop emits at most
   one `roster.update` **per 100 ms** to all authenticated connections; the
   snapshot includes the receiving station itself (§6.8).
4. `RosterFor(stationID)` (issue 13's `SnapshotProvider`): same snapshot
   serialization; station id parameter reserved for future filtering — v1
   returns the full roster for everyone.
5. `station.status` handling: payload `{mic:"ok"|"blocked"}` validated
   strictly; anything else → `error bad_message` (via Sender), no state
   change. Mic state resets to "unknown" on reconnect until re-reported
   (client sends it right after `auth.ok` — noted in issue 20).
6. Revoked stations disappear from the roster in the same broadcast that
   follows the revocation hook.
7. Implementation notes: single mutex over roster state; broadcast loop is
   one goroutine started/stopped via the Runner wiring; no store writes.

## Acceptance Criteria

- [ ] Connect/disconnect of B is visible in A's next `roster.update` within
      200 ms (fake clock: exactly one coalesced update for a burst of 5
      changes inside 100 ms).
- [ ] `station.status mic:blocked` reflects in everyone's roster; malformed
      payload → `bad_message` error and no broadcast.
- [ ] Rename via hook changes the name in the next snapshot; revoke removes
      the entry and its connection.
- [ ] BusyProvider flip + Invalidate → busy=true propagates.
- [ ] Snapshot includes self; entries sorted by case-folded name (stable UI).
- [ ] No goroutine leaks after stop (goleak).

## Validation

`go test -race ./internal/signal/...` with fake ws.Sender collectors, fake
clock, goleak.

## Dependencies

13.

## Non-goals

Call state machine (15), busy computation (16), admin polling endpoint
(12 — consumes the same provider).

## Design References

DESIGN.md §6.8 (roster.update, station.status), §6.9.
