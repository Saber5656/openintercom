# 08: HTTP server skeleton and security middleware

## Summary

`internal/httpserver`: the HTTPS app listener and plain-HTTP setup listener,
the full middleware chain (headers, CSP, rate limiting, limits, logging), the
REST envelope, static-asset serving with SPA fallback, and `/healthz`.

## Context

DESIGN §6.4 fixes headers, envelope, and middleware order; §9.7 fixes rate
limits. Handlers from later issues mount into the router this issue builds.

## Scope

- `internal/httpserver/` (server.go, middleware.go, ratelimit.go, envelope.go,
  static.go, healthz.go)
- `serve` wiring (Runner from issue 03), `--dev-web-proxy`

## Detailed Requirements

1. Two `http.Server`s per §6.4. HTTPS: `TLSConfig{MinVersion: tls.VersionTLS12,
   GetCertificate: caManager.GetCertificate}`. HTTP listener only if
   `listen_http != ""` (routes arrive in issue 09; until then: healthz +
   placeholder 302). Timeouts on both: `ReadHeaderTimeout 5s`,
   `ReadTimeout 30s`, `WriteTimeout 30s` (WS endpoint later opts out via
   per-route `http.ResponseController`/hijack — leave a documented hook),
   `IdleTimeout 120s`, `MaxHeaderBytes 16KiB`.
2. Middleware chain, outermost first (§6.4): panic-recovery (500 envelope,
   stack at `error` level, never leaks stack to client) → request-id (16 hex,
   response header `X-Request-Id`) → access-log (§6.14 allowlist: id, method,
   path-without-query, status, bytes, duration, remote IP) → security headers
   (§6.4 table **verbatim**, incl. Cache-Control split: `no-store` default;
   `static.go` overrides for hashed assets) → rate limiter → body limit
   (`http.MaxBytesReader` 64 KiB on JSON routes; 413 envelope) → router.
3. Rate limiter: token-bucket per key with background GC; config per §9.7
   table via a declarative `RouteLimit{Pattern, PerIP rate.Limit, Burst,
   GlobalPerHour int}` registry; returns 429 envelope with `Retry-After`
   seconds. Buckets keyed by remote IP (no proxy header trust — hub is
   directly reached on the LAN; `X-Forwarded-For` ignored by design).
4. Envelope helpers: `WriteOK(w, data any)`, `WriteErr(w, status int, code,
   msg string)` emitting §6.4 shapes; error codes limited to the §6.4 enum
   (typed constants). 415 on wrong Content-Type for JSON routes; 405 with
   `Allow` header from the mux.
5. Static serving (`static.go`): serve `internal/webassets.FS` (placeholder
   `index.html`/`admin/index.html` until issue 19): exact-file hit → serve
   with immutable caching iff filename carries a content hash
   (`.[0-9a-f]{8,}.` pattern); `/admin` and `/admin/*` without extension →
   `admin/index.html`; other extensionless GET → `index.html`; both
   `no-store`. Correct `Content-Type` via extension map.
6. `--dev-web-proxy URL`: reverse-proxy every non-`/api`, non-`/healthz` path
   to the Vite dev server, only when the flag is set; refuse the flag unless
   `log_level=debug` to keep it out of production habit.
7. `/healthz` on both listeners per §6.4 (no auth, no store access — pure
   liveness).
8. Router: stdlib `http.ServeMux` (Go 1.22 patterns, e.g.
   `POST /api/v1/pair`). Route registration API for later issues:
   `Register(mux Routes)` where each route declares `AuthLevel` (public |
   station | admin) — enforcement middleware itself lands in issues 10/11,
   but the declaration field is created **now** so issue 29 can introspect
   the full table.

## Acceptance Criteria

- [ ] Every response (200, 404, 405, 413, 415, 429, 500, static, healthz)
      carries the §6.4 security headers — one table-driven test iterates all.
- [ ] CSP string equals §6.4 byte-for-byte.
- [ ] Panic in a handler → 500 envelope, no stack in body, request logged.
- [ ] Rate limit: 6th pair-route request in a minute from one IP → 429 +
      `Retry-After`; different IP unaffected; global cap enforced.
- [ ] Body >64 KiB → 413 envelope. Wrong Content-Type → 415.
- [ ] SPA fallback: `/anything` → index.html, `/admin/x` → admin shell,
      `/api/nope` → 404 JSON envelope (never HTML).
- [ ] TLS listener serves the CA-issued leaf (handshake test pinned to CA).

## Validation

`go test -race ./internal/httpserver/...` (httptest + real TLS listener on
:0); manual `curl -kv` spot-check of headers.

## Dependencies

03, 04, 06.

## Non-goals

Business routes (09–12), WS upgrade specifics (13), auth enforcement logic
(10/11 — only the declaration hook lands here).

## Design References

DESIGN.md §6.4, §6.14, §9.6, §9.7; ADR-004.
