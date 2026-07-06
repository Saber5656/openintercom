# 11: Pairing lifecycle API and station identity endpoints

## Summary

Implement pairing end-to-end at the API level: admin creates codes
(R11–R13), a device redeems one (R1), and stations manage their own identity
(R2 `GET /me`, R3 `DELETE /me`) behind station bearer auth.

## Context

Pairing is the product's enrollment security boundary (§9.3): 8-digit
crypto-random codes, 10-min TTL, single-use, ≤5 outstanding, hashed at rest,
uniform failure responses.

## Scope

- `internal/api/pairing.go`, `internal/api/station_me.go`
- Station bearer-auth middleware (`AuthLevel=station`)
- QR SVG generation for R11
- Events: `pairing.created`, `pairing.revoked`, `station.paired`,
  `station.self_unpaired`

## Detailed Requirements

1. `POST /api/v1/admin/pairings` (admin): generate code = 8 digits from
   `crypto/rand` (rejection-sampled, uniform); store SHA-256 + TTL
   `now+10min` via store DAO (cap-5 eviction there). Response per R11:
   `{"id","code","expires_at","qr_svg","pair_url"}` where
   `pair_url = https://<best-name>:<port>/#/pair?code=<code>` and `qr_svg`
   encodes `pair_url`. **Plaintext code exists only in this response.**
   QR: use `github.com/skip2/go-qrcode` to get the module bitmap, emit
   minimal SVG (one `<rect>` per dark module, `shape-rendering=
   "crispEdges"`, `viewBox` sized to modules+quiet zone) — no other dep.
2. `GET /api/v1/admin/pairings` (admin): list without codes (R12).
   `DELETE /api/v1/admin/pairings/{id}` (admin): revoke unused (R13);
   deleting a used/expired entry → 404.
3. `POST /api/v1/pair` (public; §9.7 limits): body
   `{"code","name","icon"}`. Validate: code `^[0-9]{8}$`; name per §9.6
   (UTF-8, NFC normalize, trim, 1–24 chars, no control chars, case-folded
   uniqueness); icon ∈ preset list
   `kitchen, living, dining, bedroom, kids, study, entrance, garage,
   workshop, generic` (constant shared with the frontend via
   `web/src/lib/protocol.ts` — single hand-maintained list, cross-checked by
   test fixture committed in both trees). Consume code atomically (store
   `Consume`); create station (UUIDv4, token via issue 07); respond R1 shape
   (token plaintext appears only here). Failures: any code problem →
   400 `pairing_failed` (indistinguishable, §9.3); duplicate name →
   409 `conflict`; validation → 400 `invalid_request` naming the field.
4. Station bearer middleware (`AuthLevel=station`): parse
   `Authorization: Bearer oit_…`, fast-reject bad format, SHA-256 lookup via
   `Stations.GetByTokenHash`, constant-time confirm, attach station to
   request context, `TouchLastSeen` (throttled ≥60 s). 401 `unauthorized`
   otherwise.
5. `GET /api/v1/me` (station): R2 shape (station + hub name + settings
   snapshot). `DELETE /api/v1/me` (station): delete station, emit
   `station.self_unpaired`, and call the revocation hook (interface defined
   in issue 12; here: a no-op registration point so WS teardown attaches
   later without import cycles).
6. Events emitted with station id (never tokens/codes); pairing code never
   appears in any log or URL path (assert via access-log test — `pair_url`
   uses a fragment, which browsers don't send to the server, by design).

## Acceptance Criteria

- [ ] Happy path: create code → pair → station listed in store; token works
      on `GET /me`; second use of the code → `pairing_failed`.
- [ ] Expired (fake clock +11 min) and revoked codes → `pairing_failed`
      with byte-identical body to the used-code case.
- [ ] 6th outstanding code evicts the oldest unused (store behavior visible
      through R12).
- [ ] Duplicate name "kitchen" vs "Kitchen " → 409.
- [ ] Icon outside preset list → 400 `invalid_request`.
- [ ] `qr_svg` decodes (test with a Go QR decoder or golden-file compare) to
      `pair_url`; URL uses `#` fragment for the code.
- [ ] Bearer middleware: malformed, unknown, and revoked tokens → 401; `/me`
      of station A never returns station B data.
- [ ] Rate limit: 6th `POST /pair` from one IP in a minute → 429.

## Validation

`go test -race ./internal/api/...` end-to-end over httptest TLS with real
store; log-capture assertion that no 8-digit sequence from issued codes
appears in logs.

## Dependencies

05, 07, 08, 10.

## Non-goals

Onboarding UI (21), admin pairing UI (27), WS auth (13 — same token, separate
surface).

## Design References

DESIGN.md §6.7 R1–R3/R11–R13, §9.2, §9.3, §9.6, §9.7.
