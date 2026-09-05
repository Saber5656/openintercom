# 32: User documentation (English)

## Summary

The complete English user-facing docset: README overhaul, install guide,
per-OS device setup guides, troubleshooting, FAQ — good enough that a
stranger self-hosts v1 without reading source code.

## Context

DESIGN §13. Docs are a release gate (ISSUE_PLAN §1 item 4). The CA-trust
step is the highest-friction moment of the product (ADR-002); these docs
carry it.

## Scope

- `README.md` overhaul; `docs/user/{install,setup-ios,setup-android,
  troubleshooting,faq}.md`
- CONTRIBUTING release/dev-workflow refresh

## Detailed Requirements

1. `README.md`: product promise paragraph (grandma-test language);
   3-step quickstart (run hub → set admin password → pair a phone);
   feature/limitation table (LAN-only, audio-only 1:1, floor iOS 15/
   Android 9); security stance summary (§9.8 commitments verbatim:
   no cloud, no recording, media P2P-encrypted, hub never sees audio;
   link SECURITY.md); screenshots/GIF placeholders with a tracked TODO
   to replace from the §12.4 run; badges (CI, release, license); doc
   links table; "the old phone is the point" framing per ADR-006 —
   docked, powered, screen on.
2. `install.md`: paths for (a) bare binary + systemd on Debian/RPi OS —
   copy-paste blocks incl. user creation, dirs, unit install, first
   `admin set-password`; (b) Docker/compose — incl. `--network host`
   note for mDNS (issue 18) and volume; (c) macOS/Windows dev-ish run.
   Post-install checklist: static IP/DHCP reservation recommendation
   (§10), open the setup URL, expected `cert info` output.
3. `setup-ios.md` / `setup-android.md`: numbered walkthrough with
   screenshot slots per step — reach setup page → download CA →
   **iOS two-step trust** (profile install + Certificate Trust Settings
   toggle, with the "why" paragraph) / **Android CA install** (settings
   path per version range + honest explanation of the "network may be
   monitored" warning and our name-constraints containment, linking a
   short "what can this CA actually do" box) → open station app →
   pairing (QR or typed code) → mic permission → sound → keep-awake
   settings (§7.6 exact paths) → dock it. Each guide ends with a
   10-second self-test (call another station).
4. `troubleshooting.md`: symptom-first sections: "call connects then
   'couldn't connect'" → AP/client isolation explanation + router
   checklist (U3); "can't reach openintercom.local" → IP fallback + mDNS
   notes (U2); "no ring sound" → autoplay unlock path, media volume vs
   ring volume; "mic blocked" per OS; "certificate warning came back" →
   IP changed / cert rotated / wrong network; "station shows offline
   overnight" → keep-awake settings; "browser unsupported" → Firefox-
   Android note. Each section: cause → fix → verify.
5. `faq.md`: privacy (what the hub can/can't see — §9.8), "is installing
   a CA safe?" (honest, name-constraints explained), battery health for
   permanently-docked phones, "can I use it away from home?" (VPN
   answer, ADR-005), "does it record?" (no, structurally), device
   floor rationale, contributing pointer.
6. Style: plain English, second person, no unexplained jargon; every
   claim consistent with DESIGN (reviewer cross-checks §9.8/§3.2);
   relative links valid (`lychee` or equivalent link check added to CI
   docs job — extend issue 02's workflow with a docs lint job here).
7. CONTRIBUTING: dev quickstart refresh (`make dev` flow with
   `--dev-web-proxy`), test pyramid map (which suite lives where),
   release process pointer (31).

## Acceptance Criteria

- [ ] A fresh-eyes reader (not the author — use a second agent or human)
      executes install.md + one setup guide against a clean hub and
      reaches a working two-station call, logging every stumble; stumbles
      fixed or ticketed.
- [ ] Every §12.4 checklist step has a corresponding documented
      instruction (cross-reference table in the PR description).
- [ ] Link check green in CI; markdownlint (or equivalent) clean.
- [ ] Screenshot slots enumerated with exact required content notes
      (populated post-§12.4; placeholders acceptable to merge).
- [ ] README security stance matches SECURITY.md (34) verbatim where
      overlapping.

## Validation

Fresh-eyes walkthrough log attached to PR; CI docs job green.

## Dependencies

09, 25, 31 (documents final flows/artifacts; drafting may start earlier).

## Non-goals

Japanese translation (33), screenshots capture itself (release-time task),
docs website/SSG (v2 — plain GitHub markdown is v1).

## Design References

DESIGN.md §13, §9.8, §7.6, §10; ADR-002, ADR-005, ADR-006; ISSUE_PLAN
U2/U3.
