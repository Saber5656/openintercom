# 07: Auth primitives: tokens, admin password, sessions

## Summary

`internal/auth`: station bearer tokens, argon2id admin password, admin session
+ CSRF tokens, and the `admin set-password` CLI — the building blocks for §9.2.

## Context

Every authenticated surface (REST, WS, admin UI) builds on these primitives.
Formats are contract: `oit_`/`ois_` prefixes, SHA-256 at rest, PHC argon2id.

## Scope

- `internal/auth/` (tokens.go, password.go, sessions.go, csrf.go)
- CLI `admin set-password`

## Detailed Requirements

1. Tokens: `NewStationToken() (plaintext string, sha256hex string)` —
   `oit_` + base64url(no padding) of 32 bytes from `crypto/rand` (43 chars
   after prefix); `HashToken(plaintext) string`; `ValidStationTokenFormat`
   (prefix + length + alphabet) used to fast-reject garbage before hashing.
   Same pair for admin sessions with `ois_` prefix.
2. Password hashing: argon2id via `golang.org/x/crypto/argon2`,
   params `t=1, m=64*1024 KiB, p=4, salt=16B, key=32B`; encode/parse standard
   PHC string `$argon2id$v=19$m=65536,t=1,p=4$<b64salt>$<b64key>`.
   `HashPassword(pw)`, `VerifyPassword(pw, phc) (bool, error)` — verify must
   parse params from the PHC string (forward-compatible), compare via
   `subtle.ConstantTimeCompare`.
3. Sessions (uses store DAO from 05): `CreateSession(now) (cookieValue, csrf,
   Session)` TTL 24 h; `ValidateSession(cookieValue, now)` — lookup by
   SHA-256, expiry check, **sliding renewal**: if `expires_at - now < 23h`,
   extend to `now+24h` (write-behind at most once per hour per session).
   `DeleteSession`. CSRF: 32-byte random hex stored in the session record;
   `CheckCSRF(session, header)` constant-time.
4. Password length rule 8–128 chars enforced in `HashPassword` callers'
   validation helper `ValidateNewPassword`.
5. CLI `openintercom admin set-password`: interactive TTY prompt (no echo,
   `golang.org/x/term`), confirm twice, validate, write PHC to store; if
   stdin is not a TTY, require `--stdin` and read exactly one line (for
   automation/tests). Refuses to run while the hub holds the store lock —
   surface the store's lock error verbatim plus "stop the hub first".
6. Nothing in this package logs any input or derived secret; test asserts no
   `slog` calls exist in the package (grep-based lint in test).

## Acceptance Criteria

- [ ] Round-trip: `HashPassword` → `VerifyPassword` true; wrong password
      false; tampered PHC → error (not false-positive).
- [ ] Known-vector test: fixed salt (injected reader) reproduces a golden PHC
      string — protects against silent param drift.
- [ ] Token format validator rejects wrong prefix/length/alphabet; hashes are
      64 lowercase hex.
- [ ] Session expiry honored; sliding renewal extends at most once/hour
      (fake clock).
- [ ] `admin set-password --stdin <<< "hunter22"` then login verify succeeds
      (integration-lite test using a temp store).

## Validation

`go test -race ./internal/auth/...`; fuzz `VerifyPassword` PHC parser
(malformed inputs must error, never panic).

## Dependencies

03, 05.

## Non-goals

HTTP wiring (10/11), rate limiting (08), password strength estimation beyond
length, multi-admin.

## Design References

DESIGN.md §6.1, §9.2, §9.3, §9.9; ADR-004.
