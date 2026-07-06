# 17: Signaling integration tests with scripted stations

## Summary

Black-box integration suite: boot the real server (TLS, store, API, WS,
signal core), drive it with scripted station clients over real sockets, and
prove every §6.10 path end-to-end minus actual media.

## Context

Issues 05–16 are each unit-tested; this suite is the contract test that the
assembled hub honors DESIGN §6.7/§6.8 as one system — and it becomes the
regression net for every future server change. It is also the reference
implementation of the protocol for frontend issue 20.

## Scope

- `internal/integration/` test-only package (`//go:build integration` not
  required — keep in default `go test` run; it's fast)
- Test harness: `StartHub(t)` + `ScriptedStation` client helper

## Detailed Requirements

1. `StartHub(t) *Hub`: temp data dir; config with `listen_https: "127.0.0.1:0"`,
   `listen_http: "127.0.0.1:0"`, mdns off; admin password set via the real
   CLI code path (`admin set-password --stdin`); returns base URLs + an
   `http.Client`/WS dialer trusting the generated CA (read `ca.crt` from the
   data dir — also validates ADR-002 end-to-end). Full cleanup via
   `t.Cleanup` (goleak-checked).
2. `ScriptedStation`: thin typed client — REST (`Pair`, `Me`, `SelfUnpair`)
   and WS (`Connect` performing auth, channel of decoded envelopes, helpers
   `ExpectType(t, "call.state", timeout)` with pretty diffs, senders for
   every C→S type). No retry logic — failures must fail loudly.
3. Admin helper: `AdminClient` (login with cookie jar + CSRF header,
   `CreatePairing`, `Stations`, `Revoke`, `PutSettings`, `Events`).
4. Scenarios (separate test funcs; each fully independent via fresh hub —
   or shared hub + fresh stations where safe; prefer fresh hub, it's <1 s):
   - **Happy call**: pair A,B → both WS online → A invite B → both ringing
     views → B accept → connecting → A sends offer (dummy SDP string 2 KiB)
     → B receives byte-identical → answer back → ICE ×3 each way + null
     end-marker → both send `call.media connected` → active → A hangup →
     both `ended/hangup`; admin Events shows started/answered/ended with
     duration ≥0.
   - **Decline**, **Cancel**, **Timeout** (set `ring_timeout_s:10` via admin
     first; fake-clock not available over the wire — accept a 10 s real wait
     in this one test, mark it `testing.Short`-skipped).
   - **Busy**: C invites B mid-call → `error busy`; roster shows B busy=true
     via a third station's `roster.update`.
   - **Glare**: A→B and B→A racing (fire both inviting goroutines) → exactly
     one session rings, other side gets `busy` (assert eventual consistency:
     one `ringing` pair + one error).
   - **Mid-ring disconnect** both directions (close WS abruptly).
   - **Revoke mid-call**: admin deletes B → A gets `ended/error`, B's socket
     closes 4401, B's token dead on REST (401), roster shrinks.
   - **Resync**: kill A's socket during active (TCP close, no hangup) →
     reconnect+auth within grace → `auth.ok.active_call` carries the session
     → call still `active` (B saw nothing terminal).
   - **Roster propagation**: rename via admin → all stations receive updated
     name ≤2 s.
5. Assertion hygiene: every `call.state` compared as a struct (role, peer id,
   state, reason) — not substring matching; timeouts 5 s default via helper.

## Acceptance Criteria

- [ ] All scenarios green under `go test -race`; total runtime <60 s
      (short mode <20 s, timeout test skipped).
- [ ] Suite fails (demonstrated once during development, then reverted) when
      a deliberate bug is introduced into the router — proves sensitivity.
- [ ] goleak clean per test; no fixed ports, parallel-safe (`t.Parallel()`
      across scenario funcs).
- [ ] CI (issue 02 `go-test` job) runs this by default with no extra config.

## Validation

`go test -race ./internal/integration/... -count=2` (flake check);
`-short` verified.

## Dependencies

11, 16 (assembled server).

## Non-goals

Real WebRTC/media (30), performance/load testing, fuzzing (29).

## Design References

DESIGN.md §6.7, §6.8, §6.10, §10, §12.1.
