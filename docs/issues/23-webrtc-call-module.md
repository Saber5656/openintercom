# 23: WebRTC call module

## Summary

`web/src/lib/call.ts`: the media engine — getUserMedia acquisition,
RTCPeerConnection lifecycle with fixed roles (caller offers), trickle ICE
via the signaling channel, connection watchdog, mute, and idempotent
teardown.

## Context

DESIGN §7.4/§7.5; ADR-003 (host-only ICE, P2P DTLS-SRTP). This module is
UI-free and socket-free: it consumes/produces typed signaling payloads via
callbacks, so it is unit-testable with mocked WebRTC APIs and reusable by
the call UI (24).

## Scope

- `lib/call.ts` (`CallSession` class), `lib/audio-io.ts` (remote `<audio>`
  element manager)
- vitest suite with `RTCPeerConnection`/`getUserMedia` mocks

## Detailed Requirements

1. Public surface:

   ```ts
   interface CallIO {                         // wired to StationSocket by 24
     sendOffer(sdp: string): void; sendAnswer(sdp: string): void;
     sendIce(candidate: RTCIceCandidateInit | null): void;
     sendMediaConnected(): void;
     onFatal(reason: 'mic_denied'|'media_failed'): void;  // → call.hangup upstream
   }
   class CallSession {
     constructor(role: 'caller'|'callee', io: CallIO, opts?: {rtc?: typeof RTCPeerConnection, gum?: typeof navigator.mediaDevices.getUserMedia});
     start(): Promise<void>;                  // caller: gUM→offer; callee: waits for offer
     handleOffer(sdp: string): Promise<void>; // callee path
     handleAnswer(sdp: string): Promise<void>;
     handleIce(c: RTCIceCandidateInit | null): Promise<void>;
     setMuted(m: boolean): void; get muted(): boolean;
     connectionState(): string;               // for the UI indicator
     stats(): Promise<{audioBytesReceived: number}>; // E2E probe (30)
     close(): void;                           // idempotent full teardown
   }
   ```
2. Config: `new RTCPeerConnection({ iceServers: [] })` (ADR-003 — constant,
   not configurable in v1). gUM constraints exactly §7.5:
   `{audio: {echoCancellation:true, noiseSuppression:true,
   autoGainControl:true}, video:false}`.
3. Ordering rules (§7.4):
   - Caller `start()`: gUM → addTrack → createOffer → setLocalDescription →
     `io.sendOffer`. gUM rejection → `io.onFatal('mic_denied')`, no pc
     created.
   - Callee: `start()` arms the session; on `handleOffer`:
     setRemoteDescription → gUM → addTrack → createAnswer →
     setLocalDescription → `io.sendAnswer`. (Remote first, so ICE
     candidates arriving early can be buffered against the pc.)
   - ICE candidates arriving before the remote description are queued
     internally and flushed after `setRemoteDescription` resolves.
   - `onicecandidate` → `io.sendIce(candidate ?? null)` (null = end marker,
     relayed per §6.8).
   - `ontrack` → attach stream to the singleton audio element
     (`audio-io.ts`: one `<audio autoplay playsinline>` appended to body,
     reused across calls, volume 1.0 — ring volume does not affect voice).
   - `connectionstatechange === 'connected'` (first time) →
     `io.sendMediaConnected()`.
4. Watchdog: `failed` immediately, or `disconnected` continuously >10 s →
   `io.onFatal('media_failed')` once; owner (24) sends `call.hangup` and
   tears down. No ICE restart in v1.
5. `close()`: stop all local tracks, `pc.close()`, detach stream, clear
   queues/timers; safe to call multiple times and after failures at any
   stage (guard every async continuation with a `closed` flag — no
   setLocalDescription-after-close errors in console).
6. `setMuted`: toggles `track.enabled` on local audio tracks only.
7. No renegotiation: `negotiationneeded` ignored; extra inbound offers
   ignored with console.warn (§6.11 notes server relays them; client is
   tolerant).
8. Stats probe: `getStats()` summed `inbound-rtp` audio `bytesReceived` —
   used by E2E (30) and the diagnostics screen.

## Acceptance Criteria

- [ ] Mocked happy paths: caller emits offer→(answer applied)→ICE both ways
      → `sendMediaConnected` on connected; callee mirror-image with
      remote-first ordering asserted (call-order recorded by mocks).
- [ ] Early-ICE queue: candidates delivered before `handleOffer` completes
      are applied after SRD, in order, none lost.
- [ ] gUM deny: caller and callee paths both surface `mic_denied`, create no
      dangling pc, and `close()` remains safe.
- [ ] Watchdog: fake timers — `disconnected` 9.9 s → nothing; 10.1 s →
      exactly one `media_failed`; `failed` → immediate.
- [ ] Teardown idempotency: `close()` ×3 at every lifecycle stage (property
      test over stages) → no throws, tracks stopped once.
- [ ] Mute toggles `enabled` without renegotiation; state readable.
- [ ] Two real browser tabs against the dev hub complete an audible call
      (manual gate before merge; automated in 30).

## Validation

vitest with hand-rolled WebRTC mocks (committed under `web/src/test/rtc-mocks.ts`,
reused by 24/30 component tests); coverage ≥90% of `call.ts`; manual
two-tab call.

## Dependencies

19 (types/scaffold). Consumed by 24; exercised against the real server via
16 in E2E (30).

## Non-goals

UI (24), TURN/ICE servers (v2), renegotiation/ICE-restart, video, SDP
inspection/munging (§9.6 — opaque by design).

## Design References

DESIGN.md §7.4, §7.5, §6.8, §6.11, §10; ADR-003;
research/webrtc-pwa-support-old-devices.md (mDNS-candidate nuance U8).
