# 21: Onboarding wizard UI

## Summary

The six-step station onboarding wizard (§7.2): language → connection check →
pairing code+name+icon → mic priming → sound unlock → keep-awake guidance,
ending with a paired, ready station.

## Context

This is the make-or-break UX for non-technical installs (US-2). Every step
must be skip-proof (a station that looks ready must actually be able to
ring: §7.5). Server contract: R1 (issue 11).

## Scope

- `views/onboarding/` (wizard shell + 6 step components), wizard state
  machine in `state.ts`
- `lib/audio.ts` first slice: `unlockAudio()`, `playChime()`
- Deep-link handling for `#/pair?code=XXXXXXXX` (QR flow)

## Detailed Requirements

1. Entry condition: no `oi.token` in storage → app boots into onboarding;
   also reachable via "Forget this station" (issue 22). If the URL fragment
   contains `#/pair?code=…`, prefill and jump to step ③ (fragment cleared
   from history immediately after read — code never hits the server in a
   URL, §9.3).
2. Step ① Language: auto-detected, two big buttons EN/日本語 (persist
   `oi.locale`), skippable by tapping continue.
3. Step ② Connection check: if `location.protocol !== 'https:'` → full-stop
   card linking to the setup page (`http://<host>:8080/`) with explanation;
   else GET `/healthz` and show hub reachability; failure → troubleshooting
   hints (same-Wi-Fi check, IP retry).
4. Step ③ Pair: 8-digit code input (large per-digit boxes, numeric inputmode,
   paste-friendly), station name field with preset room chips (i18n'd
   labels for the 10 preset ids from `presets.json`) + free text (24-char
   counter), icon auto-follows chip with manual grid override. Submit → R1
   via `lib/api`. Errors mapped: `pairing_failed` → "code didn't work —
   check with the person who set up the hub, codes expire after 10 minutes";
   `conflict` → name-specific message with rename hint; `rate_limited` →
   wait message with countdown. Success → persist token+station atomically,
   proceed.
5. Step ④ Mic priming: explainer ("this room's phone needs its microphone"),
   button triggers `getUserMedia({audio:true})` then immediately stops all
   tracks; grant → record mic=ok; deny → per-OS recovery instructions
   (UA-sniffed iOS/Android variants, i18n) with a re-try button; allow
   proceeding only after grant **or** an explicit "continue without mic —
   this station can't talk" escape hatch that flags mic=blocked (station
   remains useful as a caller-target… no: callee needs mic too — escape
   hatch labels the limitation honestly: "you can fix this later in
   Settings"; roster will show the warning, §6.9).
6. Step ⑤ Sound unlock: big "Enable sound" button → `unlockAudio()`
   (resume AudioContext) + `playChime()`; sets `oi.soundUnlocked`. The step
   cannot be skipped (a mute station is a broken intercom).
7. Step ⑥ Keep-awake: per-OS instruction cards (§7.6 content:
   iOS Auto-Lock Never + optional Guided Access; Android charging/keep-on
   options), "docked & plugged in" framing (ADR-006), finish button →
   home screen; WS client (20) starts and sends mic status.
8. Wizard state machine as a typed reducer (step, per-step data, error) —
   unit-testable without DOM; back navigation allowed except from ⑥→③ after
   pairing succeeded (token exists; back exits to home).
9. Every screen honors the design tokens (≥64 px targets); portrait-first.

## Acceptance Criteria

- [ ] Fresh profile → wizard completes against a dev hub end-to-end; token
      stored; home renders; station appears in another device's roster.
- [ ] QR deep link prefails: `#/pair?code=12345678` lands on ③ with code
      filled and fragment stripped from the address bar.
- [ ] All R1 error codes render their distinct messages (mock API tests).
- [ ] Mic deny path shows OS-correct instructions and allows recovery
      without restarting the wizard.
- [ ] Reload mid-wizard resumes at the correct step (pre-token steps restart
      at ①③ boundary: language kept, form kept in memory only — document
      that a reload before submit loses the typed code by design).
- [ ] Sound-unlock step blocks continuation until tapped; chime audibly
      plays (manual matrix item).
- [ ] vitest: reducer transitions, error mapping, deep-link parse, i18n keys
      exist for every string (i18n:check).

## Validation

vitest suite + manual run against dev hub on one iOS and one Android device
(screenshots in PR); Playwright covers the flow later (30).

## Dependencies

19, 20.

## Non-goals

Roster/home rendering (22), ringtones beyond the unlock chime (24), PWA
install prompts (25), setup *page* (09 — server-side, pre-trust).

## Design References

DESIGN.md §7.2 (onboarding), §7.5, §7.6, §7.9, §9.3; ADR-006;
research/webrtc-pwa-support-old-devices.md.
