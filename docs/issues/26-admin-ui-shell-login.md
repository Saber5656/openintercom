# 26: Admin UI shell and login

## Summary

The `/admin/` app skeleton: login screen, session bootstrap, CSRF plumbing,
401 handling, navigation frame — the platform the two admin feature issues
(27/28) plug into.

## Context

DESIGN §8. The admin UI is the installer's only surface (US-1). It is a
separate Vite entry (ADR-004) so station devices never download admin code.
Server contract: R5–R7 (issue 10).

## Scope

- `web/src/admin/` (entry, router, layout, auth store)
- Admin API client wiring (CSRF header, 401 interceptor)
- Login + not-configured screens

## Detailed Requirements

1. Boot sequence: `GET /api/v1/admin/session` → 200: store `csrf`, render
   app; 401: render Login. Login submit → R5; map errors:
   `unauthorized` → "wrong password" inline; `admin_not_configured` →
   dedicated card rendering the exact command
   `openintercom admin set-password` in a copyable code block with a short
   explanation; `rate_limited` → cooldown message with countdown.
2. Admin API client (extends 19's `lib/api.ts` admin variant): attaches
   `X-OI-CSRF` from the auth store to every non-GET; on any 401 response →
   clear auth store → Login screen (single interceptor; no per-call
   handling); on 403 CSRF failure → transparent one-shot recovery: refetch
   R7, retry once, then surface error.
3. Layout: left nav (desktop) / top tabs (narrow): Stations, Pairing,
   Settings, Events, Security — routes `#/stations` (default), `#/pairing`,
   `#/settings`, `#/events`, `#/security`; header shows hub name (from R14,
   fetched post-login) + hub version chip (R19) + Logout (R6 → Login).
   Route stubs render "coming in 27/28" placeholders until those merge.
4. i18n: reuses the same catalogs/namespacing `admin.*` keys (en+ja).
5. Desktop-first responsive (breakpoint 720 px); still fully usable on a
   phone (the installer may only have phones — the point of the product).
6. Idle behavior: no auto-logout beyond the 24 h server session; on session
   expiry mid-use the 401 interceptor lands on Login without losing the
   current hash route (post-login returns to it).
7. Security posture: no admin data cached in localStorage (memory only);
   logout clears store; the CSRF token never appears in URLs or storage
   (memory only, refetched via R7 on reload).

## Acceptance Criteria

- [ ] Cold load unauthenticated → Login; correct password → Stations stub;
      reload keeps the session (cookie) and re-arms CSRF via R7 without
      re-login.
- [ ] `admin_not_configured` card shows on a fresh hub and disappears after
      `set-password` + login (manual against dev hub).
- [ ] Kill the session server-side (restart with cleared sessions) →
      next action lands on Login, then returns to the same route after
      login.
- [ ] Mutating call without CSRF (forced in test) recovers via the R7
      retry path exactly once, then errors visibly.
- [ ] Rate-limit path renders countdown; wrong password ×3 shows inline
      error each time.
- [ ] vitest: auth store reducer, interceptor logic (mocked fetch),
      route guard; i18n:check green.

## Validation

vitest + manual flows against dev hub (fresh + configured states).

## Dependencies

19, 10.

## Non-goals

Stations/pairing features (27), settings/events/security panels (28),
multi-admin, HTTPS-vs-HTTP handling (admin is HTTPS-only by construction —
served from the app origin).

## Design References

DESIGN.md §8, §6.7 R5–R7/R19, §9.2.
