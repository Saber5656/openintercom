# ADR-003: Hub-authoritative signaling over WSS; peer-to-peer DTLS-SRTP media, host-only ICE

Date: 2026-07-06
Status: accepted

## Context

A home deploys 2–8 stations on one LAN with an always-on hub (product decision).
The call needs a signaling channel (discovery, ringing, SDP exchange) and a media
path. Alternatives: hub relays media (SFU-style), full mesh signaling without a
hub, or hub-signaling + P2P media.

## Decision

- **Signaling**: every station holds one WSS connection to the hub. The hub owns
  the roster and a **server-authoritative call state machine** (DESIGN.md §6.10);
  clients are dumb renderers of pushed state. The hub relays SDP/ICE blobs between
  exactly the two parties of an active session and never interprets them.
- **Media**: direct peer-to-peer WebRTC (DTLS-SRTP) between the two stations.
  `iceServers: []` — host candidates only, which suffice on a single home subnet.
  The hub never sees or stores audio.
- **No STUN/TURN in v1.** The known failure mode (Wi-Fi AP/client isolation
  blocking P2P) is detected via ICE failure and answered with a targeted
  troubleshooting message. A hub-embedded TURN relay is the designed v2 escape
  hatch and must not be blocked by v1 interfaces.

## Consequences

- Server-authoritative state = one place for busy/glare/timeout rules; weak
  implementation agents get an exhaustive transition table instead of distributed
  consensus bugs.
- Media privacy is structural (hub never touches audio) — a marketable security
  property for an OSS intercom.
- Households with AP isolation cannot use v1 without router configuration;
  documented limitation.
- Hub restart drops signaling sessions (in-memory only); clients reconnect and
  reset to idle. Acceptable for a home appliance; documented in §10.
