# 34: Security policy and disclosure docs

## Summary

Publish the project's security posture: SECURITY.md (reporting, scope,
commitments), a public-facing threat-model digest, and a verification
checklist that the repo's scanning stack (from issue 02) is actually
armed.

## Context

DESIGN §9 is the internal model; an OSS release needs the public contract:
how to report vulnerabilities, what's supported, what the product promises
(§9.8) and refuses (ADR-005). GitHub repo settings themselves are
owner-managed; this issue delivers docs + a verification checklist, not
settings changes.

## Scope

- `SECURITY.md`, `docs/security-model.md` (public digest)
- Repo-scanning verification checklist (in the PR description / issue
  comment, executed with the owner)

## Detailed Requirements

1. `SECURITY.md`:
   - Reporting: GitHub Security Advisories (private) as the channel; no
     email required v1; acknowledgment target ≤7 days, fix-or-plan
     communicated ≤90 days; credit policy.
   - Supported versions table (latest minor only during v1.x).
   - **Deployment warnings, verbatim commitments**: never expose
     `:8443`/`:8080` to the internet (ADR-005 wording); the CA-trust
     model and what a leaked `data/` implies (CA key!); remote access =
     your own VPN.
   - Scope: what counts as a vulnerability here (auth bypass, CSP/CSRF
     break, token leak, cert mis-issuance, media reaching the hub, DoS
     below the §9.7 documented floor) vs. not (LAN flooding, physical
     access to an unlocked station, user removing name constraints).
   - Signing/checksums status (checksums now; cosign = roadmap).
2. `docs/security-model.md`: 1–2 page digest of DESIGN §9 for users and
   reviewers: diagram of trust boundaries (reuse §4 mermaid, simplified),
   the five §9.8 commitments, residual-risk table (TOFU, user-CA,
   LAN-DoS) in honest plain language, link to full DESIGN §9. This is the
   page README's security section links to.
3. Cross-file consistency: README (32) security summary, SECURITY.md and
   security-model.md must not drift — single source rule: commitments
   live word-for-word identical in SECURITY.md and are quoted elsewhere;
   consistency asserted by a docs-lint grep script (extend
   `scripts/docs-i18n-check.mjs` sibling `scripts/docs-consistency.mjs`).
4. Scanning verification checklist (executed with repo owner; results
   pasted into the issue):
   `gh api repos/{o}/{r}` + web UI checks — CodeQL runs green on main;
   Dependabot alerts + security updates enabled; secret scanning + push
   protection enabled; private vulnerability reporting enabled; branch
   protection/ruleset on main active. Any gap → owner action item
   (agent does not change repo settings).
5. Japanese note: SECURITY.md stays English (GitHub convention);
   README.ja.md links it with one ja sentence of guidance (coordinate
   with 33).

## Acceptance Criteria

- [ ] SECURITY.md + security-model.md merged; linked from README and
      from the admin UI Security panel caption (28 — text-only link
      addition, coordinate if 28 already merged).
- [ ] Commitments identical across files (consistency script green;
      demonstrated failing on a scratch edit).
- [ ] Private vulnerability reporting verified reachable (test advisory
      draft created and discarded, or UI screenshot).
- [ ] Scanning checklist fully executed; each item ✔ or an owner action
      item filed.
- [ ] Threat-model digest reviewed against DESIGN §9 for accuracy (no
      overclaiming — especially around E2E encryption wording: DTLS-SRTP
      peer-to-peer, hub relays SDP, hub compromise = active-MITM risk,
      §9.1 wording reused).

## Validation

Docs-consistency script in CI; checklist evidence in the issue; wording
review pass.

## Dependencies

01, 02 (scanning stack must exist to verify).

## Non-goals

Changing GitHub org/repo settings (owner-only), bug bounty program, cosign
signing implementation (roadmap note only), pentest engagement.

## Design References

DESIGN.md §9 (esp. §9.1, §9.8, §9.10–§9.11); ADR-002, ADR-005;
ISSUE_PLAN §1.
