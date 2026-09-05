# ADR-005: v1 is LAN-only — no cloud, no WAN listener, remote access delegated to user VPN

Date: 2026-07-06
Status: accepted

## Context

An intercom could be reachable from outside the home (answer the door while out).
That requires some combination of: public exposure or relay infrastructure, real
certificates, TURN, stronger authentication, and an abuse story for an
internet-facing audio device. This is the single largest security-scope decision
of the product.

## Decision

v1 is **LAN-only by design**:

- The hub binds to LAN interfaces and is never intended to be port-forwarded;
  docs explicitly warn against exposing `:8443`/`:8080` to the internet.
- No cloud components, no accounts, no telemetry, no external network calls at
  runtime (the binary functions on an air-gapped LAN).
- Households that want remote access are pointed at **their own VPN**
  (WireGuard/Tailscale) in the docs; over a VPN the client is effectively on the
  LAN and the app works unchanged. This is documentation, not code.
- The threat model (DESIGN.md §9.1) therefore assumes LAN-resident adversaries
  (guest devices, compromised IoT), not internet-scale attackers — while still
  applying defense-in-depth (TLS everywhere, token auth, rate limits) so a later
  WAN mode doesn't start from zero.

## Consequences

- Massive scope and attack-surface reduction for v1; secure default posture for
  an OSS release.
- "Answer from outside" is not a v1 feature; recorded as a v2 candidate with its
  own future security design (real certs, TURN, per-person identity).
- SECURITY.md must state plainly: exposing the hub to the internet is unsupported
  and dangerous.
