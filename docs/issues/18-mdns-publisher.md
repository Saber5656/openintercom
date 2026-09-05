# 18: mDNS advertisement (best-effort)

## Summary

`internal/mdns`: advertise `<mdns_name>.local` → hub addresses so iOS
devices reach the hub by name; strictly best-effort, feature-flagged,
never fatal.

## Context

DESIGN §6.13. Name-based access matters because the TLS story is nicer with
names, and iOS resolves `.local` natively; Android reliability is a known
unknown (U2) which is why every UX surface also shows the IP.

## Scope

- `internal/mdns/` publisher (Runner), config wiring (`advertise_mdns`,
  `mdns_name`)
- Library selection note

## Detailed Requirements

1. Library candidates: `github.com/hashicorp/mdns`, `github.com/pion/mdns`,
   `github.com/grandcat/zeroconf`. Selection criteria (record the outcome as
   a comparison table in a code-adjacent `README.md` in the package):
   pure Go, can **answer A/AAAA queries for a custom host name** (not just
   service discovery), maintained, license compatible. If none cleanly
   supports plain host publishing, publish a service instance
   `_openintercom._tcp.local` *plus* host A record via the chosen lib's
   host-entry support (hashicorp/mdns `MDNSService` with `HostName` covers
   this).
2. Publisher Runner: starts on `serve` when `advertise_mdns: true`; answers
   for `<mdns_name>.local` with all current private IPv4s (same source as
   issue 06's SAN helper — extract shared `internal/netutil`); clean
   shutdown on ctx cancel.
3. Failure policy: any error (socket busy, no multicast route, container
   without host networking) → `slog.Warn` once with remediation hint
   ("use the IP URL / host networking"), subsystem stays down, hub healthy.
4. Docker note for docs (32): mDNS requires `--network host` (Linux) —
   surface this in the package README and the install guide stub.
5. IP change handling: re-announce when the IP set changes (poll via a 60 s
   ticker comparing the netutil snapshot; cheap).
6. No config = defaults on (per §6.2): a fresh hub is reachable at
   `https://openintercom.local:8443` on iOS out of the box when the network
   allows it.

## Acceptance Criteria

- [ ] Unit: Runner starts/stops cleanly (goleak), warn-and-continue on a
      forced socket error (inject listener factory).
- [ ] With flag off: no sockets opened (asserted via injected factory).
- [ ] Manual verification procedure documented in the package README:
      `dns-sd -q openintercom.local` (macOS) / `avahi-resolve -n` (Linux)
      returns the hub IP; executed once and pasted into the PR description.
- [ ] IP-set change triggers re-announce (fake netutil provider).
- [ ] CI-safe: network-touching test guarded by
      `OPENINTERCOM_TEST_MDNS=1` env; default run is hermetic.

## Validation

`go test -race ./internal/mdns/...` (hermetic parts); manual dns-sd check on
a real LAN recorded in the PR.

## Dependencies

03, 04.

## Non-goals

DNS-SD service *browsing* (no consumer in v1), IPv6 records (v1 IPv4-first
per §6.5 — revisit with U2), Windows hub mDNS quirks beyond warn-and-continue.

## Design References

DESIGN.md §6.13, §6.5 (shared IP enumeration), §10; ISSUE_PLAN U2.
