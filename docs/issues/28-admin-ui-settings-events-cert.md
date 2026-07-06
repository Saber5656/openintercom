# 28: Admin UI: settings, events, security panel

## Summary

The remaining admin screens: runtime settings form, the event log browser,
and the security panel (CA fingerprint/download, cert status, leaf
rotation, hub info).

## Context

DESIGN §8 (Settings, Events, Security). Server contracts R14–R19 (issue
12). Completes the admin surface for v1.

## Scope

- `admin/views/settings/`, `admin/views/events/`, `admin/views/security/`

## Detailed Requirements

1. Settings (R14/R15): form with `hub_name` (text, 1–32, live counter) and
   `ring_timeout_s` (numeric stepper 10–120, step 5, unit label "seconds",
   helper text "how long a call rings before giving up"); dirty-state
   tracking (save disabled until changed, warn on route leave while dirty);
   save → PUT full object → success toast; 400 renders field-level errors.
   Note under ring timeout: "applies to new calls".
2. Events (R16): newest-first list; each type rendered as a human sentence
   with icon (i18n templates per §6.12 type list, e.g. `call.ended` →
   "{caller} → {callee}, {duration}"); station ids resolved to names via
   the stations list (unknown id → "removed station"); relative timestamps
   with absolute tooltip; "Load more" using `next_before_seq` until null;
   refresh via the shared poll scheduler (30 s here); empty state text.
   Unknown event types render raw type string (forward compatibility).
3. Security panel:
   - CA block: SHA-256 fingerprint in grouped monospace with copy button +
     "must match the setup page and `openintercom cert info`" caption;
     CA download link (`/api/v1/ca.crt`); CA expiry date.
   - Leaf block (R17): SAN list chips, `server_not_after` with color
     states (>90 d normal / <90 d amber / <30 d red "will auto-renew on
     next hub restart");
   - Rotate action (R18): button + confirm dialog ("stations stay
     trusted — the CA does not change; brief connection blips possible")
     → success shows the new expiry.
   - Hub info block (R19): version, uptime (humanized), listeners,
     stations online.
4. All three screens: loading skeletons, error banners via the shared
   poll/backoff utility, no layout shift on refresh (fixed row heights).
5. No new endpoints, no client-side persistence of any admin data.

## Acceptance Criteria

- [ ] Settings: out-of-range and boundary values (9/10/120/121) behave per
      validation on both client preview and server response; dirty-guard
      prompts on navigation; saved ring timeout visible to a station on
      its next call (manual: shorter ring observed).
- [ ] Events: 120 seeded events page correctly ×3 "Load more"; names
      resolve; removed-station fallback shown after a revoke; unknown type
      renders raw.
- [ ] Security: fingerprint matches `cert info` output character-for-
      character (manual); rotate flow updates `server_not_after` without
      admin re-login and stations stay connected (manual with dev hub —
      brief WS reconnect acceptable, roster returns green ≤30 s).
- [ ] CA download from this panel installs successfully on a test phone
      (manual matrix cross-check).
- [ ] vitest: settings form reducer/validation, event sentence renderer
      per type (table-driven over §6.12 list), pagination reducer.
- [ ] i18n en/ja complete (i18n:check), including every event template.

## Validation

vitest + manual pass against dev hub (settings round-trip, rotation, event
browsing after a scripted call session from issue 17's helper).

## Dependencies

26, 12.

## Non-goals

Log export/download (v2), event filtering/search (v2), CA rotation from UI
(CLI-only by design, ADR-002), metrics/dashboards (v2).

## Design References

DESIGN.md §8, §6.7 R14–R19, §6.12; ADR-002.
