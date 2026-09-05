# 24: Call UI: incoming, outgoing, active + ringtones

## Summary

The full-screen call experience: outgoing ("calling…"), incoming
(ring + Accept/Decline), active call (mute/hang-up/duration), ended
interstitial — wired to the server's authoritative `call.state`, the
CallSession media module, and WebAudio ringtones.

## Context

This is US-3, the product's core moment. Clients are dumb renderers of
`call.state` (§6.8/§6.10); this issue is the choreography layer between
store, `StationSocket` (20) and `CallSession` (23), plus §7.5 audio.

## Scope

- `views/call/` (Outgoing, Incoming, Active, Ended overlays)
- Call orchestrator in `state.ts` (store slice + effects)
- `lib/audio.ts` completion: ringtone/ringback/end patterns, master gain
- Wake-lock hold during calls (via lib/wake)

## Detailed Requirements

1. Orchestrator (single source of truth, driven only by store events):
   - Outbound: home tile tap → send `call.invite` → on `call.state ringing
     (role=caller)` show Outgoing + start ringback loop. Cancel button →
     `call.cancel`.
   - Inbound: `ringing (role=callee)` → show Incoming full-screen (raise
     over everything incl. settings), start ringtone loop at `oi.ringVolume`
     + vibration pattern `[400,200]` loop where supported. Accept →
     send `call.accept` (UI moves to "connecting" immediately on the
     resulting `call.state`); Decline → `call.decline`.
   - `connecting`: both sides construct `CallSession` (role from
     `call.state`), wire `CallIO` to socket sends
     (`webrtc.offer/answer/ice`, `call.media`), route inbound `webrtc.*`
     envelopes into the session; caller `start()` fires gUM→offer; callee
     `start()` then `handleOffer` on arrival. Show "connecting…" state.
   - `active`: stop all tones; show Active screen (peer name/icon, duration
     from `answered` timestamp, mute toggle bound to `session.setMuted`,
     hang-up → `call.hangup`, connection dot from
     `session.connectionState()`).
   - `ended(reason)`: teardown session (`close()`), stop tones/vibration,
     show Ended interstitial 2 s with i18n reason text:
     declined → "declined", timeout → caller "no answer" (callee side
     handled by missed logic in 22), canceled → callee silent dismiss /
     caller none, busy → "busy", connect_failed/error → "couldn't connect"
     + troubleshooting link, peer_disconnected → "connection lost",
     hangup → simple "call ended". Then return to home.
   - `onFatal` from CallSession: `mic_denied` → send `call.hangup`, show
     mic-recovery card (reuse 21's); `media_failed` → send `call.hangup`
     (server reasons it as hangup; local interstitial shows
     "couldn't connect" — deviation note: reason text local-overridden for
     accuracy).
2. Robustness rules: all UI transitions keyed by `session_id` — stale
   `call.state` for an unknown/old session ignored; `error busy` /
   `peer_unavailable` responses to an invite render as toast on home (no
   overlay); double events idempotent; app reload during a call: on
   `auth.ok.active_call` ≠ null → immediately send `call.hangup`
   (client lost media context; §6.10 note) and show "call ended".
3. Ringtone patterns (lib/audio, WebAudio, master GainNode =
   `oi.ringVolume`):
   - incoming: 880 Hz 400 ms + 660 Hz 400 ms, 800 ms gap, loop; gentle
     5 ms attack/release envelopes (no clicks).
   - ringback (caller): 440 Hz 1000 ms on, 2000 ms off, loop.
   - end cue: 520→380 Hz two-tone 150 ms each, once.
   - chime (from 21): ascending triad 120 ms notes.
   All generated via OscillatorNode; `stopAll()` hard-stops on any state
   exit (belt: also on `visibilitychange hidden`… no — ringing must
   continue if screen blanks; only stop on state exit).
4. Wake lock: request on `ringing|connecting|active`, release on `ended`
   (progressive, lib/wake).
5. Accept/Decline ergonomics: buttons ≥ 88 px tall, 24 px+ apart, Accept on
   the right (thumb-reach), pressed-state feedback <100 ms, both debounced.
6. Vibration: `navigator.vibrate` guarded (Android only), stops on state
   exit.

## Acceptance Criteria

- [ ] Two-device manual: full US-3 loop audible both ways; ring stops
      crisply on accept/decline/cancel/timeout on **both** sides.
- [ ] vitest: orchestrator reducer — every `call.state` × current-UI
      combination maps to the specified screen/tone (table-driven, includes
      stale-session ignores and reload-resync hangup).
- [ ] Ringtone scheduling with mocked AudioContext: correct
      frequencies/timing/loop + `stopAll` on exit paths; volume follows
      `oi.ringVolume`.
- [ ] `error busy` invite response → home toast, no overlay flash.
- [ ] Mute round-trip visible in UI state and `track.enabled` (mock).
- [ ] Reason texts localized (i18n:check) and match §6.10 reasons 1:1
      (exhaustive switch — `tsc` exhaustiveness via `never` guard).
- [ ] No lingering oscillators/wake locks after 20 rapid call cycles
      (leak assertion via mock registries).

## Validation

vitest suites; manual two-phone session (iOS+Android) recorded in PR;
Playwright automation lands in 30.

## Dependencies

20, 22, 23.

## Non-goals

Missed-badge bookkeeping (22), server timing rules (16), custom ringtone
uploads (v2), speaker/mic device pickers (v2).

## Design References

DESIGN.md §7.2 (call screens), §7.4, §7.5, §6.8, §6.10; ADR-006.
