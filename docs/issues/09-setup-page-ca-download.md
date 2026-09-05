# 09: Setup page and CA download (HTTP listener)

## Summary

The plain-HTTP bootstrap surface: a dependency-free bilingual setup page, the
CA download, fingerprint display, and redirect-to-HTTPS — DESIGN §6.6.

## Context

First contact happens before the device trusts our CA, so this is the only
deliberately-HTTP surface. It serves public bytes and instructions, nothing
else. TOFU risk is mitigated by showing the CA fingerprint for comparison
with `openintercom cert info`.

## Scope

- HTTP-listener routes: `GET /`, `GET /ca.crt`, catch-all redirect
- HTTPS-listener route: `GET /api/v1/ca.crt`
- Embedded static setup page (HTML/CSS only, en+ja)

## Detailed Requirements

1. `GET /` (HTTP listener): static HTML+CSS, **zero JavaScript**, served from
   `go:embed`. Language: `?lang=ja|en` query param wins; else pick by
   `Accept-Language` prefix; page shows `EN | 日本語` links (query-param
   switch). Content blocks, in order: what OpenIntercom is (one paragraph);
   step list ① download CA → ② install/trust per-OS (two collapsible-by-CSS
   sections: iOS with the two-step trust warning highlighted, Android with
   the "network may be monitored" explanation); ③ **CA SHA-256 fingerprint**
   rendered in a large monospace block with the sentence "must match
   `openintercom cert info` on the hub"; ④ continue link to
   `https://<best-name>:<https-port>/`.
   `best-name` = `hostname` if configured, else `<mdns_name>.local` **and**
   the primary private IPv4 shown side by side (Android users need the IP —
   research doc). Template values injected server-side (`html/template`,
   auto-escaped).
2. `GET /ca.crt`: DER bytes from `ca.Manager.CADER()`;
   `Content-Type: application/x-x509-ca-cert`;
   `Content-Disposition: attachment; filename="openintercom-ca.crt"`;
   `Cache-Control: no-store`.
3. Any other path on the HTTP listener: `302` to
   `https://<best-name>:<https-port>/`.
4. `GET /api/v1/ca.crt` on the HTTPS listener: same DER response (R4).
5. Page must render acceptably at 320 px width with default fonts (it runs on
   ancient browsers; no webfonts, no images except an inline SVG logo).
6. All strings for this page live in the page template pair (en/ja) — not in
   the frontend i18n system (this page is server-rendered).

## Acceptance Criteria

- [ ] `curl http://hub:8080/` returns HTML containing the fingerprint from
      `cert info` and both name+IP HTTPS links; `?lang=ja` switches language.
- [ ] `curl http://hub:8080/ca.crt | openssl x509 -inform der -noout` parses;
      headers as specified.
- [ ] `curl -v http://hub:8080/whatever` → 302 to the HTTPS origin.
- [ ] `GET /api/v1/ca.crt` over HTTPS returns identical bytes.
- [ ] Disabling the listener (`listen_http: ""`) removes the surface; app
      still fully works for already-trusted devices.
- [ ] Page contains no `<script>` (asserted by test).

## Validation

`go test -race` handler tests (fingerprint injection, lang negotiation, DER
equality, redirect); manual browser check on a phone-sized viewport.

## Dependencies

08 (mounts routes), 06 (CA bytes/fingerprint).

## Non-goals

The PWA onboarding wizard (21), per-OS *screenshots* (user docs, 32/33 — the
page carries text instructions and links to the docs site section).

## Design References

DESIGN.md §6.6, §9.1 (TOFU), §9.5; ADR-002.
