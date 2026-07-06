# Research: WebRTC / PWA capabilities on the target "old smartphone" floor

Status: informs [ADR-001](../decisions/ADR-001-client-pwa-over-native.md), [ADR-003](../decisions/ADR-003-hub-topology-and-p2p-media.md), [ADR-006](../decisions/ADR-006-stations-are-dedicated-terminals.md)
Date: 2026-07-06

## Support floor (decided with the product owner)

Devices up to ~8 years old:

| Platform | Floor | Example devices | Browser situation |
|---|---|---|---|
| iOS | iOS 15 | iPhone 6s / 7 / 8 / SE1-2 (6s & SE1 top out at 15.x) | Safari is frozen with the OS. iOS 15 Safari is the *capability floor* of the whole product. |
| Android | Android 9 | 2017–2018 mid/high-end (Pixel 2/3 era) | **Chrome updates via Play independently of the OS**, so old Android generally runs a *current* Chrome. Bottlenecks are hardware (mic/speaker quality) and OS power management, not web APIs. |

Consequence: design decisions must be checked against **iOS 15 Safari** first;
Android is comparatively easy.

## Capability matrix at the floor

Confidence: high for standardized behavior, medium where marked (verify on real
devices during the frontend wave; tracked as a known unknown).

| Capability | iOS 15 Safari | Android 9 + current Chrome | Design consequence |
|---|---|---|---|
| `getUserMedia` (audio) | ✅ | ✅ | Core is viable. |
| WebRTC unified-plan, Opus | ✅ | ✅ | Audio-only 1:1 calls fine. |
| Built-in echo cancellation (AEC) | ✅ (quality varies by device — medium) | ✅ | Stations are speakerphones; rely on browser AEC (`echoCancellation: true`), no custom DSP in v1. |
| WebSocket | ✅ | ✅ | Signaling channel fine. |
| Screen Wake Lock API | ❌ (arrived in Safari 16.4) | ✅ | Cannot rely on Wake Lock at the floor → keep-awake is an *OS-settings instruction*, not an API guarantee (see below). |
| Web Push | ❌ (16.4+, installed PWA only, via Apple push service = cloud) | ⚠️ requires a push service (cloud) | **No push in v1** — incompatible with LAN-only + floor. Ringing works only while the app is open in the foreground → the "dedicated terminal" product stance (ADR-006). |
| Service Worker / add-to-home-screen | ✅ | ✅ | PWA shell OK. Note: iOS may evict script-writable storage for sites unused ~7 days; stations are used daily, and re-pairing is cheap. |
| AudioContext autoplay | 🔒 needs a user gesture once | 🔒 same | Onboarding includes an explicit "enable sound" tap that unlocks audio; re-shown after reload if the context is suspended. |
| Vibration API | ❌ | ✅ | Vibration is progressive enhancement only. |

## Keep-awake strategy (no Wake Lock at the floor)

Stations are docked, powered, screen-on appliances. The reliable mechanism at the
floor is **device settings, not web APIs**:

- iOS: Settings → Display & Brightness → Auto-Lock → **Never**; optionally Guided
  Access to pin Safari full-screen.
- Android: "Keep screen on while charging" (developer options) or a long screen
  timeout; optionally app/screen pinning.
- Where Wake Lock *is* available (Android, iOS ≥16.4) the app requests it as a
  progressive enhancement and re-acquires it on `visibilitychange`.

The onboarding wizard shows per-OS instructions; the docs treat "phone in a dock,
always powered" as the supported deployment.

## ICE on a home LAN

- Both stations sit on the same subnet → **host candidates suffice; no STUN/TURN**
  in v1 (`iceServers: []`).
- mDNS candidate obfuscation: browsers hide host IPs behind `.local` names **only
  when getUserMedia permission has not been granted**. In-call we always hold mic
  permission, so real host candidates are exchanged and same-subnet connectivity is
  direct. (Confidence: medium-high; covered by the E2E suite.)
- Known failure mode: Wi-Fi **AP/client isolation** (common on guest SSIDs) blocks
  peer-to-peer frames → ICE fails with host-only candidates. v1 answer: detect ICE
  failure, show a targeted troubleshooting message. Hub-embedded TURN relay is the
  designed v2 escape hatch (ADR-003).

## Codec/quality defaults

- Opus mono, default bitrate (~32 kbps effective), `echoCancellation`,
  `noiseSuppression`, `autoGainControl` all `true`.
- No SDP munging in v1. No renegotiation (audio-only, fixed roles: caller offers).

## Verification plan

Automated E2E runs on desktop Chromium with fake media devices (CI). The floor
itself is validated by a **manual device matrix checklist** (DESIGN.md §12.4) run
before release on at least one iOS 15/16 device and one Android 9+ device:
mic permission flow, CA trust flow, ring audibility from lock-adjacent states,
call audio quality/AEC, overnight WS reconnect stability.
