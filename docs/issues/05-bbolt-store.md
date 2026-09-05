# 05: Embedded data store (bbolt)

## Summary

`internal/store`: bbolt-backed persistence with the §6.3 bucket schema, typed
DAOs, schema versioning, and capped event storage.

## Context

Single source of durable state (stations, pairings, settings, admin, events).
Callers never touch bbolt types. Plaintext secrets never stored (§9.3, §9.9).

## Scope

- `internal/store/` — `store.go` (open/close/migrate), `stations.go`,
  `pairings.go`, `settings.go`, `admin.go`, `events.go`, tests

## Detailed Requirements

1. `Open(dir string) (*Store, error)`: file `<dir>/openintercom.db`, mode
   `0600`, `bbolt.Options{Timeout: time.Second}`; on lock timeout return an
   error explaining "another hub instance is using this data directory".
   Create all buckets; initialize `meta/schema_version=1`; refuse versions >1
   with an upgrade-required error; run forward-only migrations otherwise.
2. Value encoding: JSON via per-type structs matching §6.3 exactly
   (field names are contract — snake_case). Timestamps RFC 3339 UTC.
3. DAO surface (all methods take no bbolt types; each is one transaction):
   - `Stations`: `Create(st Station) error` (conflict if id exists or
     case-folded name duplicate), `Get(id)`, `GetByTokenHash(hex)`, `List()`,
     `Rename(id, name)` (same duplicate rule), `SetIcon(id, icon)`,
     `TouchLastSeen(id, t)`, `Delete(id)`.
   - `Pairings`: `Create(p Pairing) error` enforcing **≤5 unused**: if a 6th
     unused would exist, delete the oldest unused first. `Consume(codeHash,
     now) (Pairing, error)`: single transaction — find by code hash, reject if
     used or expired, mark `used_at=now`; exactly-once under concurrency.
     `List()`, `Delete(id)`, `PruneExpired(now)`.
   - `Settings`: `HubName() string`, `SetHubName`, `RingTimeout() int`,
     `SetRingTimeout` — validation per §9.6 (1–32 chars; 10–120) enforced here
     *and* at the API layer; unknown settings impossible by construction.
   - `Admin`: `PasswordHash() (string, bool)`, `SetPasswordHash(phc string)`;
     `Sessions` DAO: `Create(hash, sess)`, `Get(hash)`, `Delete(hash)`,
     `PruneExpired(now)`.
   - `Events`: `Append(ev Event) (seq uint64, err error)` — key = 8-byte
     big-endian next-seq; after insert delete oldest while count>500.
     `ListDesc(limit int, beforeSeq *uint64) ([]Event, nextBefore *uint64)`.
4. No goroutines, no logging of values (keys/hashes only at debug), no
   plaintext token/code/password parameters anywhere in this package —
   callers pass hashes (enforced by naming + review).
5. `Close()` flushes and releases the file lock.

## Acceptance Criteria

- [ ] All DAO methods round-trip with exact §6.3 JSON field names (golden
      tests marshal and compare).
- [ ] `Consume` under 50 concurrent goroutines on one code → exactly 1 success.
- [ ] Events cap: appending 600 leaves exactly 500, oldest gone, `ListDesc`
      pagination stable across the prune.
- [ ] Second `Open` on a locked dir fails within ~1 s with the explanatory error.
- [ ] `schema_version=2` file → open fails with upgrade-required error.
- [ ] DB file mode `0600` on disk.

## Validation

`go test -race ./internal/store/...` with `t.TempDir()`; include a small
benchmark for `Events.Append` at cap (sanity, not a gate).

## Dependencies

01.

## Non-goals

Backup/restore tooling (v2); business rules beyond storage constraints
(pairing TTL policy decisions live in issue 11; the store only checks
expiry timestamps it's given).

## Design References

DESIGN.md §6.3, §6.12, §9.3, §9.6, §9.9; ADR-004.
