# 33: User documentation (Japanese)

## Summary

Japanese translations of the complete user docset plus a Japanese README,
kept structurally in lockstep with the English originals.

## Context

The product's first real household is Japanese-speaking; the OSS audience
is global. English stays canonical (issue 32); Japanese is a first-class
translation with a defined sync process.

## Scope

- `docs/user/ja/{install,setup-ios,setup-android,troubleshooting,faq}.md`
- `README.ja.md` + cross-language links

## Detailed Requirements

1. Translate all five user docs from issue 32 into natural Japanese
   (no machine-translation tone): です・ます調, technical terms kept in
   English where that's the norm (WebRTC, mDNS, CA は「認証局(CA)」初出
   説明), UI strings quoted exactly as the ja i18n catalog renders them
   (cross-check `web/src/i18n/ja.json` — mismatches are bugs, fix the
   catalog or the doc).
2. `README.ja.md`: full translation of README.md; both READMEs link to
   each other at the top (`English | 日本語`).
3. Structural lockstep: same headings/anchors (English anchor slugs kept
   via explicit `<a id>` where GitHub's auto-anchors would differ);
   screenshot slots reference the same images (screenshots are
   language-neutral where possible; per-OS settings screenshots may need
   ja-device variants — mark which).
4. Sync process (documented at the top of each ja file as an HTML
   comment): `Translated-From: <en file> @ <git commit sha>`; CI check
   (extend the docs job): if an en user doc changes without its ja
   counterpart's `Translated-From` sha being updated, warn (not fail) on
   the PR — script `scripts/docs-i18n-check.mjs`.
5. Japan-specific notes where genuinely helpful (e.g., typical home
   router client-isolation menu names by major JP vendors in
   troubleshooting — best-effort table, clearly marked as examples).

## Acceptance Criteria

- [ ] All five docs + README.ja.md complete; a Japanese reader completes
      install + iOS setup using only ja docs (fresh-eyes run as in 32).
- [ ] UI string quotes match the ja i18n catalog exactly (spot-check
      script or manual table in PR).
- [ ] `Translated-From` headers present and current; drift-warning CI
      script works (demonstrated by a scratch en edit).
- [ ] Cross-links EN⇄JA on every page; link check green.
- [ ] No literal-translation artifacts (reviewer: read-aloud test on
      setup-ios.md).

## Validation

Fresh-eyes ja walkthrough; CI docs job; drift-warning demo.

## Dependencies

32.

## Non-goals

Other locales, translating DESIGN/ISSUE docs (developer docs stay
English), translating the setup *page* (already bilingual, issue 09).

## Design References

DESIGN.md §13, §7.8; issue 32.
