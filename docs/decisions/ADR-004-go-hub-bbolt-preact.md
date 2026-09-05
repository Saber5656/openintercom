# ADR-004: Implementation stack — Go single binary, bbolt store, Preact+Vite PWA embedded

Date: 2026-07-06
Status: accepted

## Context

The hub targets Raspberry Pi / NAS / leftover PCs across linux/arm64, linux/armv7,
linux/amd64, macOS, Windows. Product owner chose Go. Remaining choices: HTTP
stack, persistence, WebSocket library, CLI framework, frontend stack, asset
delivery.

## Decision

| Concern | Choice | Rationale |
|---|---|---|
| Language / distribution | Go ≥1.24, `CGO_ENABLED=0`, single static binary per platform | Cross-compile matrix trivially; one-file install on RPi/NAS. |
| HTTP routing | stdlib `net/http` + Go 1.22 `ServeMux` patterns | Zero framework lock-in; method+path patterns are enough for our ~20 routes. |
| WebSocket | `github.com/coder/websocket` | Minimal, maintained, context-aware. |
| Persistence | `go.etcd.io/bbolt`, single file, bucket schema in DESIGN.md §6.3 | Tiny transactional KV beats SQL for ~10 stations; pure Go keeps CGO off. Local disk required (no network FS) — documented. |
| CLI | `github.com/spf13/cobra` | Ubiquitous; subcommands `serve`, `version`, `admin set-password`, `cert info`, `cert rotate-server`. |
| Passwords / tokens | `golang.org/x/crypto/argon2` (argon2id) for the admin password; 256-bit random bearer tokens stored as SHA-256 | Standard, GPU-resistant for the one low-entropy secret; hashing bearer tokens keeps the store non-sensitive at rest. |
| Frontend | TypeScript + Vite + Preact (+ `@preact/signals`), hash-based routing, two Vite entries (`index.html` station app, `admin/index.html` admin app) | React idioms with ~4 kB runtime — fits old devices; two entries keep admin code off station devices. |
| Asset delivery | `go:embed` of `web/dist` into the binary; `--dev-web-proxy` flag proxies to the Vite dev server during development | Hub and client always ship as one version; no separate deploy artifact. |
| QR codes | server-side `github.com/skip2/go-qrcode`, SVG returned inline in the pairing-create response | No client dependency; plaintext code never persisted. |

Dependency policy: the list above is close to exhaustive for the server; adding a
Go or npm runtime dependency requires an ADR note or explicit review.

## Consequences

- Weak implementation agents get a boring, well-documented stack with huge
  training coverage.
- No CGO → no sqlite; bbolt schema versioning is our own responsibility (schema
  version key + forward-only migrations, §6.3).
- Frontend bundle size and API usage must respect the iOS 15 Safari floor
  (ADR-001); CI builds enforce `tsc` targets accordingly.
