# 27: Admin UI: stations and pairing

## Summary

The two core admin screens: the live stations table (rename/revoke) and the
pairing flow (generate code → giant code + QR + countdown; manage
outstanding codes).

## Context

DESIGN §8 (Stations, Pairing). Server contracts R8–R13 (issues 11/12).
Live-ness in v1 is **polling** (no admin WS): 5 s interval while the tab is
visible.

## Scope

- `admin/views/stations/` (table, rename, revoke)
- `admin/views/pairing/` (create result view, active codes list)
- Poll scheduler utility (shared with 28's events screen)

## Detailed Requirements

1. Poll scheduler: `usePoll(fn, 5000)` — runs immediately, then every 5 s
   while `document.visibilityState === 'visible'`; pauses hidden; single
   in-flight guard; error backoff ×3 then banner "connection to hub lost —
   retrying" (auto-recovers).
2. Stations screen (R8): table/cards — icon+name, status chips
   (online/offline; busy; mic-blocked warning), last seen (relative,
   tooltip absolute), created date. Row actions:
   - Rename: inline edit → R9; 409 → inline "name already in use";
     client-side same validation preview as §9.6 (length/trim) before
     submit.
   - Remove: confirm dialog stating consequences verbatim: "This phone will
     stop working immediately and must be paired again. Any ongoing call
     will end." → R10 → optimistic row removal + refresh.
   Empty state: arrow to Pairing tab (US-2).
3. Pairing screen:
   - "New pairing code" button → R11 → **result takeover view**: 8-digit
     code in ~96 px monospace groups of 4, QR rendered from `qr_svg` via
     `<img src="data:image/svg+xml;base64,…">` (never innerHTML — §9.6),
     `pair_url` with copy button, expiry countdown (mm:ss from
     `expires_at`), "device paired!" auto-dismiss when the next stations
     poll shows a new station (nice-loop: compare id sets), Done button.
   - Below/after: outstanding codes list (R12): created, expires-in,
     used/unused; revoke (R13) with mini-confirm. Expired entries greyed
     until server prune drops them.
   - Explainer line: codes are single-use and expire in 10 minutes.
4. All times rendered client-side from RFC 3339 (no server clock
   assumptions beyond it being the same clock that minted `expires_at` —
   countdown derives from `expires_at - now` with drift note in code).
5. Accessibility: table keyboard-navigable; dialogs focus-trapped; QR has
   alt text with the pair URL.

## Acceptance Criteria

- [ ] Stations table reflects a station going online/offline within ≤6 s
      (poll cadence) — manual with dev hub + one station.
- [ ] Rename happy + 409 + validation-preview paths (vitest with mocked
      API; manual once).
- [ ] Revoke: confirm → station's phone lands on the revoked screen (22)
      and the row disappears; canceling the dialog does nothing.
- [ ] Pairing result: code/QR/URL render; QR scans with a phone camera to
      the pair deep link (manual); countdown hits 0 → view flips to
      "expired — generate a new code".
- [ ] "Device paired" auto-detect fires when pairing completes during the
      takeover view.
- [ ] Poll pauses when tab hidden (vitest fake timers + visibility mock);
      recovers after simulated 3 failures.
- [ ] i18n en/ja complete (i18n:check).

## Validation

vitest (poll scheduler, reducers, views with mocked API) + manual US-2/US-6
loop against dev hub with a real phone.

## Dependencies

26, 11, 12.

## Non-goals

Settings/events/security (28), bulk operations, station grouping (v2),
admin push updates (v1 polls).

## Design References

DESIGN.md §8, §6.7 R8–R13, §9.3, §9.6.
