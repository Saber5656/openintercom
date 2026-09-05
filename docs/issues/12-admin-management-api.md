# 12: Admin management API: stations, settings, events, cert, info

## Summary

Implement the remaining admin surface R8–R10 and R14–R19: station
list/rename/revoke, runtime settings, event log pagination, certificate
panel data + leaf rotation, and hub info.

## Context

Completes §6.7. Live presence fields (`online`, `busy`, `mic`) come from the
signal core, which lands in wave 2 — this issue defines the provider
interfaces and ships with stub implementations so the API is complete and
testable now, and issue 16 swaps in the real provider.

## Scope

- `internal/api/admin_stations.go`, `admin_settings.go`, `admin_events.go`,
  `admin_cert.go`, `admin_info.go`
- Provider interfaces: `LiveInfoProvider`, `RevocationSink`
- Events: `station.renamed`, `station.revoked`, `admin.settings_changed`,
  `cert.rotated`

## Detailed Requirements

1. Provider contracts (defined in `internal/api/providers.go`):

   ```go
   type LiveState struct { Online bool; Busy bool; Mic string } // "ok"|"blocked"|"unknown"
   type LiveInfoProvider interface { Snapshot() map[string]LiveState } // key: station id
   type RevocationSink interface { StationRevoked(stationID string) }  // close WS, end calls
   ```

   Default stub: empty snapshot / no-op sink. `serve` wiring passes the real
   implementations from issue 16 when available.
2. `GET /api/v1/admin/stations` (R8): store list merged with
   `LiveInfoProvider.Snapshot()`; absent id → offline/`unknown`. Sorted by
   name (case-folded).
3. `PATCH /api/v1/admin/stations/{id}` (R9): partial body `{name?, icon?}`;
   same validation rules as pairing (§9.6, shared validator function —
   single source); rename emits `station.renamed` and (once 16 lands)
   propagates to rosters via the provider side (nothing to do here beyond
   the event + store write; roster reacts to store change through issue 16's
   subscription — define `StationsChanged()` notification method on
   `RevocationSink`? No: keep one interface, rename it `SignalHooks` with
   `StationRevoked(id)` and `StationRenamed(id)` methods).
4. `DELETE /api/v1/admin/stations/{id}` (R10): delete from store, emit
   `station.revoked`, call `SignalHooks.StationRevoked` (which per §6.10 T13
   ends live sessions and closes the WS 4401). 404 on unknown id.
5. Settings (R14/R15): GET returns `{hub_name, ring_timeout_s}`; PUT accepts
   the **full** object, rejects unknown fields (strict decode), validates
   (1–32 chars; 10–120 int), persists, emits `admin.settings_changed`
   (values in `detail` — not secret). Ring-timeout change applies to new
   calls only (§6.7 note).
6. Events (R16): `GET /api/v1/admin/events?limit=50&before_seq=N` — limit
   1–200 default 50; newest-first; response `{"events":[…],
   "next_before_seq": N|null}`; invalid params → 400.
7. Cert panel (R17/R18): GET returns
   `{ca_fingerprint_sha256, ca_not_after, server_sans, server_not_after}`
   from `ca.Manager`; POST `/cert/rotate-server` calls `RotateServer()`,
   emits `cert.rotated`, returns fresh R17 payload. Rotation must be
   effective without restart (issue 06 atomic pointer) — assert via two
   handshakes in test.
8. Info (R19): `{version, uptime_s, listen_https, listen_http,
   stations_online}` (`stations_online` from provider snapshot).
9. All routes `AuthLevel=admin` (CSRF enforced by issue 10 middleware; no
   extra logic here).

## Acceptance Criteria

- [ ] R8 merges live data: with a fake provider marking one station
      online/busy, response reflects it; stations absent from snapshot are
      offline.
- [ ] Rename to a duplicate (case-folded) → 409; valid rename persists and
      emits event; hooks called with the right id (spy).
- [ ] Revoke: record gone, event emitted, hook called; second delete → 404.
- [ ] Settings PUT with unknown field → 400; out-of-range ring timeout → 400;
      valid change readable via GET and via station R2.
- [ ] Events pagination walks 120 seeded events in 3 pages with stable
      ordering and terminates with `next_before_seq: null`.
- [ ] Cert rotate: SANs equal before/after (same config), `NotAfter` moves
      forward, TLS handshake serves the new leaf immediately.
- [ ] Every route rejects station-token and anonymous callers (401/403) —
      spot-checked here, exhaustively in issue 29.

## Validation

`go test -race ./internal/api/...` httptest with real store + real CA manager
(temp dir) + fake providers/spies.

## Dependencies

05, 06, 10, 11.

## Non-goals

Admin UI (26–28), real live provider (16), event *emission* from call flow
(16), admin WS push (v1 admin UI polls).

## Design References

DESIGN.md §6.7 R8–R19, §6.10 T13, §6.12, §9.2, §9.6.
