# ADR-002: Hub-generated, name-constrained private CA for LAN TLS

Date: 2026-07-06
Status: accepted

## Context

Browsers only expose `getUserMedia`, service workers and Wake Lock in a **secure
context**. The hub has no public DNS name and must work fully offline. Options
compared in [research/secure-context-lan-https.md](../research/secure-context-lan-https.md):
private CA, Let's Encrypt + owned domain, self-signed click-through, Chrome flags,
plain HTTP. Only the private CA satisfies: works on iOS Safari **and** Android
Chrome, no cloud/account/renewal dependency, no per-browser hidden settings.

## Decision

1. On first start the hub generates a **CA** (ECDSA P-256, 10-year validity,
   `BasicConstraints CA:true, pathlen:0`, `KeyUsage certSign`) and a **leaf server
   certificate** (825-day validity) covering: `<mdns_name>.local`, the configured
   hostname (if any), and all current private IPv4 addresses as IP SANs.
2. The CA certificate carries a **critical Name Constraints** extension permitting
   only the OpenIntercom DNS names. IP addresses are intentionally left
   unconstrained (RFC 5280: an absent name-type is unconstrained) so DHCP changes
   never force device re-trust.
3. The leaf is **auto-reissued** at startup when SANs drift (IP changed) or expiry
   is <30 days. CA rotation is CLI-only and documented as destructive
   (re-trust on every device).
4. Devices trust the CA once, via a guided **setup page served over plain HTTP**
   (`:8080`) that offers the CA download, per-OS instructions, and the CA's SHA-256
   fingerprint for out-of-band comparison with `openintercom cert info` (TOFU
   mitigation).

## Consequences

- One-time ~2-minute setup per device; the single biggest UX cost of the product.
  Owned by the onboarding docs and setup page (issues 09, 32, 33).
- CA private key = highest-value secret on the hub: `0600` perms, never logged,
  never served, warning at startup if permissions are too open.
- Users are asked to install a CA — the name constraints cap the blast radius and
  the docs must explain this honestly (Android's "network may be monitored"
  warning included).
- Static IP / DHCP reservation for the hub is *recommended* (docs), not required.
