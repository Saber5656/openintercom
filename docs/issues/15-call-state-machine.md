# 15: Call session state machine (pure module)

## Summary

`internal/signal/callsm`: a pure, I/O-free implementation of the §6.10
session state machine — types, guards, the full T1–T14 transition table, and
an effects model — with exhaustive tests. The heart of the product.

## Context

Server-authoritative call logic (ADR-003) concentrated in one deterministic
module so correctness is provable by table-driven tests, and the router
(issue 16) stays a thin adapter. No goroutines, no channels, no clocks, no
network in this package.

## Scope

- `internal/signal/callsm/` — `types.go`, `table.go` (SessionTable),
  `machine.go`, exhaustive tests + fuzz

## Detailed Requirements

1. Types (exact):

   ```go
   type State string    // "ringing" | "connecting" | "active" | "ended"
   type EndReason string // declined|canceled|timeout|connect_failed|hangup|
                         // peer_disconnected|error|unavailable|busy
   type Session struct {
     ID, CallerID, CalleeID string
     State State
     Reason EndReason        // set iff State==ended
     StartedAt, AnsweredAt int64 // unix ms, supplied by caller of each op
     RelayBudget int          // starts 200, §6.11
   }
   ```
2. Input events (methods on `SessionTable`, each returns `[]Effect, error`):
   `Invite(callerID, calleeID, now)`, `Accept(stationID, sessionID, now)`,
   `Decline/Cancel/Hangup(stationID, sessionID, now)`,
   `MediaConnected(stationID, sessionID, now)`,
   `Relay(stationID, sessionID, kind RelayKind, size int, now)`,
   `Disconnected(stationID, now)`, `Reconnected(stationID, now)`,
   `Revoked(stationID, now)`, `Timer(sessionID, kind TimerKind, now)`.
   TimerKind: `ring`, `connect`, `disconnect_grace`.
3. Effects (pure data; executed by issue 16):

   ```go
   PushState{To string, S SessionView}        // call.state payload per §6.8
   ErrorTo{To string, Code string, RefSeq *int64}
   StartTimer{SessionID string, Kind TimerKind, AfterMS int64}
   CancelTimer{SessionID string, Kind TimerKind}
   RelayTo{To string, Kind RelayKind}         // router forwards the raw payload
   AppendEvent{Type string, Fields …}         // call.started/answered/missed/ended
   ```
4. Behavior exactly per §6.10 table T1–T14 and guards, plus:
   - `Invite` guards evaluated in order: unknown/offline callee (`IsOnline`
     is injected as a function dependency) → `peer_unavailable`; self-call →
     `bad_message`; caller busy or callee busy → `busy`. One live session per
     station enforced by table index `stationID → sessionID`.
   - Ring timeout duration is a parameter of `Invite` (router reads settings)
     — the SM stores it in the StartTimer effect, no settings dependency.
   - `Disconnected` during `ringing`: T6/T7 immediate end. During
     `connecting|active`: start `disconnect_grace` 15 s timer (T12), mark the
     participant absent; `Reconnected` cancels the grace timer; grace expiry
     ends `peer_disconnected` to the survivor only.
   - `Relay` decrements RelayBudget; at 0 → end session `error` (both
     notified) + AppendEvent. Relay allowed only in `connecting|active`, only
     by participants; `size > 64*1024` → end `error`.
   - Stale/foreign/duplicate operations (wrong sessionID, non-participant,
     event not valid in state — e.g. `Accept` by caller, double `Accept`) →
     single `ErrorTo{stale_session}` (or `bad_message` for role violations),
     **state unchanged** — idempotency under packet duplication.
   - Terminal push ordering: on any transition both participants receive
     `PushState` reflecting the same state; on `ended` the session is removed
     from the table after effects are emitted; a `disconnected` participant
     produces no PushState effect for that side (router can't deliver anyway
     — SM tracks presence flags set by Disconnected/Reconnected).
5. `SessionView` includes per-recipient `role` and `peer` fields (§6.8
   call.state) — generate two views per push.
6. Determinism: all methods take `now int64`; no `time.Now()` anywhere
   (enforced by a test that greps the package).

## Acceptance Criteria

- [ ] Every row T1–T14 has a dedicated test asserting: resulting state,
      exact effect list (order-sensitive golden compare), and table
      occupancy.
- [ ] Negative matrix: for each state × each event not legal in that state,
      assert `stale_session`/`bad_message` error effect and zero state
      change (generated combinatorially, not hand-listed).
- [ ] Busy/glare: A→B while B→A already `ringing` → second invite `busy`;
      two simultaneous invites to one callee → exactly one `ringing`
      (table lock is the caller's responsibility — document that
      SessionTable is **not** goroutine-safe and must be driven from one
      goroutine; asserted in docs + `-race` test in 16).
- [ ] Relay budget: 201st relay ends the session with `error`; oversize
      relay likewise.
- [ ] Fuzz: random event sequences (bounded alphabet) never panic; invariants
      hold: ≤1 live session per station; every session reaching `ended` was
      removed; no PushState to a station marked disconnected.
- [ ] 100% statement coverage of `machine.go` (enforced in CI for this
      package via `go test -cover` threshold script).

## Validation

`go test -race -cover ./internal/signal/callsm/...`; fuzz corpus committed;
coverage gate script added to Makefile (`make cover-callsm`).

## Dependencies

01 (scaffold only — start any time).

## Non-goals

Timers that actually fire (16), WS delivery (13/16), persistence (none by
design, §10).

## Design References

DESIGN.md §6.10 (table is normative), §6.11 (budget/size), §6.8
(call.state shape), §10; ADR-003.
