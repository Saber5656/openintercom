# ADR-001: Browser PWA client instead of native apps

Date: 2026-07-06
Status: accepted

## Context

The product revives old smartphones (support floor: iOS 15 / Android 9, see
[research](../research/webrtc-pwa-support-old-devices.md)) as intercom stations.
Distribution options: native iOS app, native Android APK, or a browser-delivered
web app (PWA) served by the LAN hub.

- Native iOS on old devices is effectively undistributable for an OSS home project
  (App Store review + minimum-OS churn, or per-device sideloading with 7-day
  re-signing on free accounts).
- Native Android APK sideloading is feasible but excludes every iPhone.
- A PWA served by the hub reaches both platforms with zero install channel, zero
  store dependency, and instant updates (reload).

## Decision

The station and admin clients are a **browser-delivered web app (PWA)** served by
the hub over HTTPS. Supported browsers: Safari on iOS 15+, Chrome on Android 9+.
Firefox (desktop) is best-effort for the admin UI only.

The capability floor for all frontend work is **iOS 15 Safari**. Features absent
there (Wake Lock, Web Push, Vibration) may be used only as progressive
enhancements with a documented fallback.

## Consequences

- No push notifications → stations must be foreground, screen-on appliances
  (ADR-006). Ringing is delivered over the app's live WebSocket.
- Requires a trusted HTTPS origin on the LAN → private CA subsystem (ADR-002).
- One codebase, embedded into the hub binary (ADR-004); the hub version and client
  version always ship together.
- A native Android "wrapper" (WebView/TWA kiosk) is a possible v2 convenience, not
  a v1 deliverable.
