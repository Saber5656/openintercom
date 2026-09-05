# 10: Admin session API: login, logout, CSRF

## Summary

Implement R5–R7 (`/api/v1/admin/login|logout|session`), the admin-session
middleware, cookie attributes, and CSRF enforcement for all admin routes.

## Context

DESIGN §9.2 defines the admin principal: argon2id password (primitives from
issue 07), `oi_admin` HttpOnly cookie, per-session CSRF token required as
`X-OI-CSRF` on every non-GET admin request.

## Scope

- `internal/api/admin_auth.go` + middleware in `internal/httpserver`
- Events `admin.login` / `admin.login_failed`

## Detailed Requirements

1. `POST /api/v1/admin/login` (public, rate-limited per §9.7: 5/min/IP +
   20/hour global): body `{"password": string}`. If no password hash exists in
   the store → 403 `admin_not_configured` (message tells the operator to run
   `openintercom admin set-password`). On verify success: create session
   (issue 07), set cookie `oi_admin=<ois_…>; HttpOnly; Secure; SameSite=Strict;
   Path=/; Max-Age=86400`, respond `{"ok":true,"data":{}}` **plus** the CSRF
   token in the JSON body `data.csrf` (the UI keeps it in memory). Failure:
   401 `unauthorized` (uniform; no user enumeration — there is only one user)
   + event `admin.login_failed` with remote IP in `detail`.
2. `POST /api/v1/admin/logout` (admin): delete session, expire cookie
   (`Max-Age=0`), 200 `{}`.
3. `GET /api/v1/admin/session` (admin): `{"authenticated":true,
   "csrf":"<token>"}` — lets a reloaded UI re-arm its CSRF header without
   re-login.
4. Admin middleware (applies to every route declared `AuthLevel=admin` in the
   issue-08 registry): read cookie → `ValidateSession` (sliding renewal per
   07; refresh Set-Cookie when extended) → for methods other than
   GET/HEAD/OPTIONS require header `X-OI-CSRF` equal (constant-time) to the
   session's token, else 403 `forbidden` with message "missing or invalid
   CSRF token". Missing/invalid session → 401.
5. On login success, log event `admin.login`. Never log the password, hash,
   cookie, or CSRF values (§9.9).
6. Login handler compares against a **dummy argon2id hash** when no session
   work is needed… (clarification: when password hash exists, always run the
   full argon2id verify even for absurd inputs — no length-based early exit —
   to keep timing uniform; `ValidateNewPassword` limits apply only to
   `set-password`).

## Acceptance Criteria

- [ ] Correct password → 200, cookie with exactly the specified attributes
      (asserted literally), `data.csrf` non-empty.
- [ ] Wrong password → 401 + `admin.login_failed` event; 6th attempt in a
      minute → 429.
- [ ] No hash configured → 403 `admin_not_configured`.
- [ ] Mutating admin request without `X-OI-CSRF` → 403; with stale token
      (post-logout) → 401; GET without CSRF header → allowed.
- [ ] Session survives hub restart (persisted, §6.3), expires after 24 h idle
      (fake clock), slides when active.
- [ ] `Secure` cookie flag present (tests run the real TLS listener).

## Validation

`go test -race ./internal/api/...` httptest with real store+auth; cookie-jar
based multi-request flows.

## Dependencies

07, 08.

## Non-goals

Admin UI (26); management endpoints (12); multi-admin/roles (out of scope for
v1 entirely).

## Design References

DESIGN.md §6.7 R5–R7, §9.2, §9.7, §9.9.
