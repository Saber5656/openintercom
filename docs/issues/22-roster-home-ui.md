# 22: Roster home screen and settings UI

## Summary

The station's resting screen: the roster tile grid with live presence,
connection banners, missed-call badges — plus the settings screen
(volume, language, diagnostics, forget).

## Context

DESIGN §7.2 (home, settings). The home screen is what a docked phone shows
23 hours a day; glanceability is the feature (US-3, ADR-006). Data source:
store fed by `roster.update` / connection state (issue 20).

## Scope

- `views/home/` (header, tile grid, banners), `views/settings/`
- Missed-call store logic (§7.9 `oi.missed`)
- Invite dispatch (call flow UX itself is issue 24)

## Detailed Requirements

1. Header: hub name, self station name+icon, connection dot
   (green online / amber reconnecting / red revoked-superseded).
2. Tile grid: every roster entry except self; tile shows icon, name, state:
   - offline → greyed, non-tappable, "offline" caption
   - idle → full color, tappable → dispatch `call.invite`
   - busy → amber "in a call" caption, non-tappable
   - mic=blocked → warning glyph overlay (tooltip/i18n caption)
   - missed badge: red counter from `oi.missed`, cleared on tile tap
     (clearing does not require calling — tap on offline tile with badge
     clears too).
   Grid: CSS grid auto-fit, min tile 40 vw portrait (2 columns typical),
   token-driven; ≥64 px targets everywhere.
3. Banners (stacked, priority order): `reconnecting…` (after 5 s offline,
   §7.3), `mic blocked — tap to fix` (→ mic recovery view from issue 21,
   reused), `superseded` full-screen card ("opened on another tab/window —
   use here instead" button → `socket.start()`), `revoked` full-screen card
   → forget+onboarding.
4. Missed-call logic: on `call.state ended` where `role==='callee'` and
   `reason==='timeout'` → increment `oi.missed[callerId]` with timestamp;
   surfaced also as a one-line toast. (Caller-side "no answer" UX lives in
   the call screens, issue 24.)
5. Empty roster state: friendly "no other stations yet — pair another phone"
   card with a pointer to the admin (US-2 loop).
6. Settings screen (gear from header): ring volume slider (0–100%, persists
   `oi.ringVolume`, live-preview via `playChime` at chosen volume), ring
   test button (plays the incoming pattern once at current volume — pattern
   stub until 24, chime acceptable interim), language toggle, keep-awake
   status line (`wakeState()` from lib/wake: active/unsupported + link to
   instructions view), diagnostics block (connection state, hub host, app
   version from build-time define, last WS error), "Forget this station"
   (two-step confirm → R3 → `clearAll()` → onboarding). Version + link to
   in-app licenses page not required in v1 (README covers licensing).
7. All rendering pure from store signals; zero fetches in components
   (data flows only via ws client and explicit intents).

## Acceptance Criteria

- [ ] Two dev stations: B toggling online/offline/busy/mic states renders
      correctly on A within 300 ms of the roster push (manual + vitest with
      scripted store).
- [ ] Tap idle tile dispatches exactly one `call.invite` (double-tap
      debounce 500 ms) and is a no-op for offline/busy tiles.
- [ ] Missed badge: simulated timeout increments, persists across reload,
      clears on tap; multiple callers tracked independently.
- [ ] Banner priority: revoked > superseded > mic > reconnecting (forced
      combinations in vitest).
- [ ] Forget flow: confirm → server 200 → storage empty → onboarding step ①;
      server failure (offline) → error toast, station **not** wiped
      (token still valid server-side; §7.9 note).
- [ ] Settings persist across reload; volume preview audible (manual).
- [ ] i18n:check green (all new strings in en+ja).

## Validation

vitest (reducers, badge logic, banner priority, tile state mapping) +
manual two-device session against dev hub; Playwright assertions land in 30.

## Dependencies

20, 21.

## Non-goals

Call overlays/ringtones (24), admin surfaces (26–28), roster filtering or
groups (v2).

## Design References

DESIGN.md §7.2 (home/settings), §7.3, §7.9, §6.8–§6.9; ADR-006.
