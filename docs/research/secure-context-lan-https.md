# Research: Serving a trusted HTTPS origin on a home LAN (secure-context problem)

Status: informs [ADR-002](../decisions/ADR-002-lan-tls-private-ca.md)
Date: 2026-07-06

## Why this matters

OpenIntercom's station client is a browser app. The APIs it depends on are **only
available in a secure context** (HTTPS with a certificate the browser trusts, or
`localhost`):

| API | Needed for | Secure context required |
|---|---|---|
| `getUserMedia` | microphone capture | yes (hard requirement) |
| `RTCPeerConnection` | audio transport | practically yes (gUM gates it) |
| Service Worker / PWA install | app shell caching, add-to-home-screen | yes |
| Screen Wake Lock | keep screen on | yes |
| Clipboard, Notifications | minor UX | yes |

The hub lives on a private LAN with no public DNS name, so we cannot use a normal
publicly-issued certificate out of the box. Whatever we choose defines the entire
first-run UX and a large part of the security model.

## Options considered

| # | Option | How | Verdict |
|---|---|---|---|
| 1 | **Hub-generated private CA**, per-device one-time trust install | Hub creates a CA at first boot, issues itself a leaf cert for its LAN names/IPs. User installs the CA cert on each phone once. | **Chosen.** Fully offline, no accounts, no renewal traffic, works on iOS Safari and Android Chrome. Cost: one-time ~2 min setup per device; mitigated by a guided setup page. |
| 2 | Public domain + Let's Encrypt DNS-01 + split-horizon DNS | User owns a domain, hub gets a real cert via DNS API, LAN DNS points the name at the hub. | Rejected for v1. Requires a paid domain, a supported DNS provider, API credentials on the hub, and periodic renewal — violates the "no cloud, no accounts" product stance. Viable v2 *optional* mode. |
| 3 | Self-signed leaf + per-browser "proceed anyway" exception | No CA install; user clicks through the warning. | Rejected. Click-through does not reliably unlock `getUserMedia`/service workers everywhere; iOS Safari WSS connections to untrusted certs fail without a useful error; the warning UX is unacceptable for a family product; trains users to ignore TLS warnings. |
| 4 | Chrome flag `unsafely-treat-insecure-origin-as-secure` | Per-device flag for the hub origin over plain HTTP. | Rejected. Chrome/Android only (impossible on iOS Safari), survives updates poorly, per-device hidden-flag surgery. |
| 5 | Plain HTTP | — | Impossible. `getUserMedia` is unavailable, full stop. |

## Platform behavior notes (for the chosen option)

Confidence: high unless marked. These target **old, frozen OS versions**, so the
behavior is stable; each item still gets re-verified on real devices during the
frontend wave (see Known Unknowns in ISSUE_PLAN.md).

### iOS (Safari, iOS 15+)

- Downloading the CA `.crt` in Safari triggers the **configuration profile** flow:
  Settings → "Profile Downloaded" → install.
- **Second step is mandatory and easy to miss**: Settings → General → About →
  Certificate Trust Settings → enable full trust for the CA. Without it the CA is
  installed but *not* trusted. The setup guide must show both steps with screenshots.
- Safari resolves `.local` names via Bonjour natively, so `https://openintercom.local:8443`
  works once trusted.
- Apple platforms enforce X.509 Name Constraints on user-installed CAs
  (confidence: medium-high; verify on-device during implementation).

### Android (Chrome, Android 9+)

- CA install: Settings → Security → Encryption & credentials → Install a certificate
  → CA certificate. Android shows a scary full-screen warning ("network may be
  monitored") — the setup guide must explain why this appears and what our CA can
  and cannot sign (see name constraints below).
- **Chrome (as a browser) trusts user-installed CAs** for HTTPS browsing. Native apps
  targeting API 24+ do not, by default — irrelevant for us (browser-only client).
- mDNS `.local` resolution from the browser on Android is historically unreliable
  across versions/OEMs. **Primary access method on Android is the hub's IP address**
  (e.g. `https://192.168.1.50:8443`); the cert therefore must carry IP SANs.
- Firefox for Android does not use the system CA store by default → **not a
  supported v1 browser** (documented; Chrome required on Android).

## Residual risks and mitigations

| Risk | Mitigation |
|---|---|
| User-installed CA could sign certs for *any* site if the CA key leaks | CA key never leaves the hub (`0600`, dedicated dir). CA certificate carries a **critical Name Constraints extension** permitting only OpenIntercom's DNS names (`openintercom.local`, optional configured hostname). IP SANs remain unconstrained by design (RFC 5280: absent name-type = unconstrained) so DHCP address changes don't force CA re-trust; a rogue IP-only cert cannot impersonate any DNS-named site. |
| First contact over plain HTTP (setup page + CA download) can be MITMed (TOFU) | Setup page and CLI (`openintercom cert info`) both display the CA's SHA-256 fingerprint; docs instruct comparing them. Accepted residual risk on the home LAN threat model. |
| Some legacy clients ignore Name Constraints | Documented residual risk; floor platforms (iOS 15+, current Chrome) enforce them. |
| CA re-generation invalidates trust on every device | CA validity 10 years; leaf certs are rotated independently (auto re-issued on SAN drift / expiry). CA rotation is an explicit, CLI-only, documented destructive action. |

## Decision

Option 1: hub-generated, name-constrained private CA with a guided HTTP setup page.
Details and consequences in ADR-002.
