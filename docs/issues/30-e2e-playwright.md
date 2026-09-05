# 30: End-to-end suite (Playwright, fake media)

## Summary

Browser-level E2E: a real hub binary + two Chromium contexts with fake
microphones complete pairing, presence, and full audio calls; wired into
CI as the final automated gate.

## Context

DESIGN §12.1 E2E layer. Integration tests (17) prove the server; this
proves the *product* — UI, WS client, WebRTC, service worker — against the
real binary. Real-device confidence remains the manual matrix (§12.4);
this suite is the every-PR regression net.

## Scope

- `e2e/` npm project (Playwright, TS), fixtures/helpers, specs
- CI wiring (path-filtered on PRs, always on `main`)

## Detailed Requirements

1. Harness (`e2e/fixtures.ts`):
   - `globalSetup`: `make build`; start `bin/openintercom serve` with temp
     data dir, `listen_https: 127.0.0.1:0`, `listen_http` disabled, mdns
     off; parse the ready log line for the bound port; run
     `admin set-password --stdin` with a fixture password.
   - Browser: chromium with
     `--use-fake-ui-for-media-stream --use-fake-device-for-media-stream
     --autoplay-policy=no-user-gesture-required`; contexts with
     `ignoreHTTPSErrors: true` (documented: trusting the CA in the
     browser under test is out of scope; server TLS correctness is covered
     by 06/17).
   - `pairStation(browser, name)`: admin API (request context with cookie
     +CSRF) creates a pairing code → new browser context walks the real
     onboarding wizard UI (not API shortcuts — the wizard IS under test),
     returns a `StationPage` helper (locators for tiles, overlays,
     banners).
2. Specs (each independent; hub per spec-file via worker-scoped fixture):
   - `onboarding.spec`: full wizard incl. deep-link `#/pair?code=` variant,
     wrong-code error path, duplicate-name 409 path.
   - `roster.spec`: A/B presence (close B's context → A shows offline
     ≤35 s — heartbeat bound; reopen → online); admin rename reflected on
     tiles ≤2 s; revoke → B lands on revoked screen, A's tile gone.
   - `call-happy.spec`: A calls B → B incoming overlay visible + A
     outgoing → B accepts → both reach active screen → **audio flows**:
     poll each page's diagnostics probe (`CallSession.stats()` exposed on
     `window.__oiDebug` in dev/e2e builds only — build flag from 23)
     until `audioBytesReceived` strictly increases on both sides → A
     hangs up → both home, admin Events shows started/answered/ended.
   - `call-paths.spec`: decline; cancel; busy (3rd station invites B
     mid-call → toast); timeout (admin sets `ring_timeout_s:10` first).
   - `reconnect.spec`: CDP network offline on A 10 s → banner appears →
     online → roster green again; reload A mid-active-call → returns
     home (resync hangup per §6.10) and B gets ended screen.
   - `admin.spec`: login, stations live chips, pairing takeover view
     (countdown ticking, "device paired" auto-detect), settings save,
     security panel renders fingerprint equal to `cert info` output
     (read via a tiny helper running the binary).
   - `pwa.spec`: SW registered on station entry; `/api/` requests bypass
     SW (assert via `page.route` observation); update toast appears after
     swapping in a rebuilt bundle (skip if flaky — mark `fixme` with
     issue link rather than deleting).
3. Flake policy: zero tolerated — every wait is condition-based (no bare
   timeouts >250 ms); retries=1 in CI with trace+video artifact upload on
   failure; a spec failing ≥2× in a week gets quarantined via `fixme` +
   tracking issue (process noted in e2e/README).
4. CI (extends issue 02 `ci.yml`): job `e2e` — on PRs touching
   `web/**`, `internal/{ws,signal,api,httpserver}/**`, `e2e/**`, or
   `Makefile`; always on `main` pushes; ubuntu-latest; Playwright browsers
   cached; total budget ≤8 min.
5. WebKit investigation (U5): timeboxed spike — attempt `call-happy` on
   Playwright WebKit with fake media; record findings in
   `e2e/README.md#webkit` (works → add as optional job; doesn't → document
   why + rely on manual matrix). Non-blocking either way.

## Acceptance Criteria

- [ ] All specs green locally (`npx playwright test`) ×3 consecutive runs
      and in CI on this issue's PR.
- [ ] `call-happy` proves increasing `audioBytesReceived` both directions
      (not just UI state).
- [ ] Failure artifacts (trace, video, hub log) uploaded on a forced
      failure (demonstrated once).
- [ ] Runtime ≤8 min in CI; specs parallel across workers with isolated
      hubs.
- [ ] e2e/README documents: running locally, debugging traces, flake
      policy, WebKit findings.

## Validation

CI runs; forced-failure artifact check; flake triple-run.

## Dependencies

12, 16, 21, 22, 24, 25 (and 27 for admin.spec pairing view).

## Non-goals

Real-device automation (manual matrix §12.4), performance testing, load
testing, TLS-trust UX automation.

## Design References

DESIGN.md §12.1–§12.2; ISSUE_PLAN §6, U5; issue 23 (stats probe).
