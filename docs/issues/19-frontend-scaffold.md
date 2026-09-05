# 19: Frontend scaffold: Vite+Preact, embed, i18n, design tokens

## Summary

Create the `web/` project (TypeScript strict, Vite, Preact + signals, two
entries), the shared libs (protocol types, API client, storage, i18n, wake),
the appliance design tokens, and the `go:embed` build integration that makes
`make build` ship the UI inside the binary.

## Context

DESIGN §7.1 fixes the module layout; ADR-004 the stack. Everything in waves
3–4 builds on this scaffold, and CI's `web` job (issue 02) activates the
moment `web/package.json` exists — so lint/type/test/build must be green
from this first commit.

## Scope

- `web/` project + tooling (eslint, prettier, vitest)
- `web/src/lib/{protocol,api,storage,i18n,wake}.ts`, `state.ts` skeleton,
  design tokens CSS, base components
- `internal/webassets` embed + Makefile/CI integration

## Detailed Requirements

1. Tooling: Vite (two entries: `index.html`, `admin/index.html`), TS
   `strict: true`, target `es2017` + `safari15` in `build.target`
   (iOS 15 floor, ADR-001); Preact + `@preact/signals`; vitest + jsdom;
   eslint (typescript-eslint, preact config) **plus a custom rule/ban**:
   `dangerouslySetInnerHTML` forbidden (§9.6) via `no-restricted-syntax`;
   prettier. npm scripts: `dev`, `build`, `lint`, `typecheck`, `test`,
   `i18n:check`. Package versions pinned (no `^`) — lockfile committed.
2. `lib/protocol.ts`: hand-written TS types mirroring §6.7 request/response
   bodies and §6.8 envelope+payload types, plus constants: close codes,
   error codes, icon preset list (must equal issue 11's server list — both
   sides import from a committed JSON fixture `web/src/lib/presets.json`;
   server test reads the same file, path documented there).
3. `lib/api.ts`: `fetch` wrapper — base `/api/v1`, JSON envelope unwrap
   (§6.4) into `Result<T, ApiError>`; bearer injection from storage for
   station calls; admin variant with `credentials: 'same-origin'` +
   `X-OI-CSRF` header injection (value provided by admin app state); typed
   functions for every route in §6.7 (R1–R19) even if unused yet.
4. `lib/storage.ts`: typed accessors exactly for the §7.9 keys, JSON-safe,
   `clearAll()` for the forget flow; storage events ignored (single-tab
   assumption; multi-tab handled by WS 4409).
5. `lib/i18n.ts`: `t(key, params?)` with `{placeholder}` interpolation;
   catalogs `i18n/en.json`, `i18n/ja.json`; locale = `oi.locale` ??
   `navigator.language` prefix ?? `en`; `scripts/i18n-check.mjs` fails on
   key-set mismatch between catalogs (wired into `npm run i18n:check` and
   the CI web job).
6. `lib/wake.ts`: `requestWakeLock()/releaseWakeLock()/wakeState()` no-op
   fallback where unsupported (§7.6) — lifecycle integration happens in 25.
7. Design tokens `src/styles/tokens.css`: dark theme default; CSS custom
   properties for color roles (bg, surface, text, accent, danger, success,
   warning), spacing scale, type scale (base 18 px — arm's-length
   readability), radii, and `--touch-target: 64px`; base components
   `ui/Button.tsx` (variants primary/danger/ghost, min size = touch target),
   `ui/Tile.tsx`, `ui/Overlay.tsx`, `ui/Banner.tsx` — all styled via plain
   CSS modules-free class conventions (`.oi-*`), **no inline styles**
   (CSP §6.4).
8. Embed integration: `internal/webassets/webassets.go` with
   `//go:embed all:dist` (a `dist/.gitkeep` + build-tag fallback keeps
   `go build` working before `npm run build`; decide: commit a tiny
   placeholder `dist/index.html` "build the web app" page, replaced by real
   builds; `make build-web` = `cd web && npm ci && npm run build` copying
   output into `internal/webassets/dist`). Update Makefile (`build-web`
   real now), CI web job turns active automatically (verify).
9. App shells render: station entry shows a placeholder home ("scaffold OK",
   locale-switched string) and `/admin/` its own placeholder — proving
   routing, tokens, i18n and embed end-to-end.

## Acceptance Criteria

- [ ] `make build && ./bin/openintercom serve` serves the station shell at
      `/` and admin shell at `/admin/` over HTTPS with the strict CSP —
      zero console errors/CSP violations in Chromium.
- [ ] `npm run lint|typecheck|test|build|i18n:check` all green; CI web job
      runs and passes on the PR.
- [ ] `dangerouslySetInnerHTML` usage fails lint (test fixture proves it,
      then removed).
- [ ] i18n check fails when a key is removed from `ja.json` (proved, then
      reverted).
- [ ] Bundle budget: station entry JS ≤ 60 KiB gzip at this stage (budget
      asserted by a size-check script — headroom target ≤ 150 KiB by v1).
- [ ] `--dev-web-proxy http://localhost:5173` serves Vite HMR through the
      hub (manual check documented in CONTRIBUTING).

## Validation

CI green; manual smoke on desktop Chromium + one real phone against a dev
hub (screenshot in PR).

## Dependencies

01 (02 activates web CI; 08 serves the embed — both already merged per wave
order).

## Non-goals

Real screens (21/22/24), WS client (20), service worker/manifest (25),
admin features (26–28).

## Design References

DESIGN.md §5, §6.4 (CSP), §7.1, §7.8, §7.9; ADR-001, ADR-004.
