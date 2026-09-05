# ADR-006: Stations are dedicated, docked, always-on terminals (no push notifications)

Date: 2026-07-06
Status: accepted

## Context

A phone in someone's pocket can only be rung via push notifications. At our
support floor (iOS 15 Safari) Web Push does not exist, and where it exists it
requires cloud push services — both conflict with ADR-001/ADR-005. See
[research/webrtc-pwa-support-old-devices.md](../research/webrtc-pwa-support-old-devices.md).

## Decision

The product stance: an OpenIntercom **station is an appliance** — an old phone in
a dock/stand, permanently powered, screen on, browser open on the station app.
It is *not* an app on the phone you carry.

- Ringing is delivered over the station's live WebSocket and rendered
  full-screen with WebAudio ringtone; no push, no background delivery.
- Keep-awake is achieved by OS settings (Auto-Lock Never / keep-awake-while-
  charging), with Wake Lock as progressive enhancement; the onboarding wizard
  teaches this per OS.
- The UI is designed for wall/counter distance: oversized touch targets (≥64 px
  for call actions), high contrast, minimal text, usable by children and elderly
  family members.
- Reliability follows: a station that is offline is *visibly* offline in every
  roster (grey tile), so "I called but it didn't ring" is diagnosable at a glance.

## Consequences

- Product docs and marketing must set this expectation explicitly ("give your old
  phone a job").
- Battery-health guidance (docked at 100% forever) belongs in the user docs.
- v2 may add a companion mode for carried phones (push via installed PWA on newer
  OSes, or a native wrapper) without changing the v1 architecture.
