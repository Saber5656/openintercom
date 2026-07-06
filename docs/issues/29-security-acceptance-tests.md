# 29: Security acceptance test suite

## Summary

A permanent, CI-enforced test suite that turns the DESIGN §9 security model
into executable checks: the full authorization matrix, abuse cases §9.11,
header/cookie guarantees, and fuzzing of externally-reachable decoders.

## Context

Security requirements rot unless they're tests. This suite is the
regression net that lets future contributors change handlers without
silently dropping a control. It runs in the normal `go test` job (fast
parts) with fuzz corpora as seed-regression tests.

## Scope

- `internal/sectest/` (or `tests/security/` — pick one, document) built on
  issue 17's `StartHub`/clients
- Route-authorization introspection test
- Go fuzz targets + committed corpora

## Detailed Requirements

1. **Authorization matrix, exhaustively generated** (§9.2): the issue-08
   route registry declares `AuthLevel` per route. The test walks **every
   registered route** (introspection API added in 08) and calls it as:
   anonymous, station (valid token), admin (valid session, with CSRF), and
   asserts the §9.2 matrix outcome class (2xx/401/403/405). A route missing
   a declaration, or a registered route not covered by the matrix table in
   the test, **fails the build** — new endpoints must declare their auth
   level to compile a green suite.
2. Targeted abuse cases (each its own test, §9.11):
   - Revoked station: token on REST → 401; on live WS → closed 4401 ≤2 s;
     replayed on new WS auth → `auth.err`.
   - Cross-session forgery: station C sends `call.accept`/`webrtc.offer`
     with A↔B's `session_id` → `stale_session` error, session unharmed
     (state re-verified via A/B views).
   - WS origin: missing, `http://` scheme, evil host, correct-host-wrong-
     port → all rejected pre-upgrade; every SAN-derived origin accepted.
   - CSRF: admin mutation without header / with random / with another
     session's token → 403; GET unaffected.
   - Pairing brute force: 6th `POST /pair` in 60 s per IP → 429; global
     hourly cap enforced across IPs (fake IPs via crafted RemoteAddr).
   - Login brute force → 429 + `admin.login_failed` events recorded.
   - Oversize: 65 KiB JSON body → 413; 129 KiB WS frame → close 4400;
     >64 KiB SDP relay → session ends `error`.
   - Relay budget: 201 ICE messages → session ends `error`.
   - Duplicate-name pairing → 409; self-call → `bad_message`; parallel
     invites to one callee → exactly one `ringing` + one `busy`.
   - Secrets in logs: run a full happy scenario with a log capture;
     assert no occurrence of: any issued token, pairing code, password,
     `Authorization` header value, argon2 hash, or SDP payload marker
     (plant a canary string in the SDP).
   - Headers: every REST response (success + each error class) carries the
     §6.4 header set; CSP string byte-exact; cookie attributes byte-exact
     on login.
3. Fuzzing (`go test -fuzz` targets, run as seed-corpus regression in CI;
   nightly long-fuzz optional, documented):
   - `FuzzWSEnvelope` (13's decoder), `FuzzPairRequest` (11's validator,
     incl. Unicode normalization edge cases), `FuzzSettingsPut` (12),
     `FuzzPHCParse` (07 — merge its corpus here or keep in 07; document).
     Property: never panic, never accept invalid per spec.
4. Suite must remain <90 s in CI (parallel hubs; fuzz = corpus replay
   only).
5. Document (file header comment) the mapping test-name → DESIGN §9.x for
   auditability; a `make sectest` target runs only this suite.

## Acceptance Criteria

- [ ] Matrix test fails when a route is added without declaration
      (demonstrated during development with a scratch route, then
      removed).
- [ ] Every §9.11 bullet has ≥1 named test; grep-able mapping comment
      complete.
- [ ] Log-canary test proves the access logger redaction and no-secret
      rule (§9.9).
- [ ] All fuzz targets have committed corpora incl. at least one
      previously-crashing input each (found during development) now fixed.
- [ ] CI runtime budget met; suite green ×3 consecutive runs (flake
      check).

## Validation

`make sectest` locally and in CI; deliberate-bug sensitivity check (remove
CSRF middleware → suite red) performed once and noted in the PR.

## Dependencies

10, 11, 12, 13, 16, 17.

## Non-goals

External pentest, dependency scanning (02), DoS/load testing (§9.7
non-goal), TLS protocol-level testing beyond config assertions.

## Design References

DESIGN.md §9 (all), §6.4, §6.8; ISSUE_PLAN §6.
