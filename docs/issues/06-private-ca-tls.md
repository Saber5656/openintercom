# 06: Private CA and TLS certificate manager

## Summary

`internal/ca`: generate a name-constrained home CA and the hub's leaf cert,
auto-reissue on SAN drift/expiry, expose `GetCertificate` for the TLS listener
and the `cert info` / `cert rotate-server` CLI.

## Context

Browsers require a trusted HTTPS origin for `getUserMedia` (ADR-002,
research/secure-context-lan-https.md). This subsystem is the root of the
product's transport security; DESIGN §6.5 is the exact spec.

## Scope

- `internal/ca/` (manager, generation, SAN computation, fingerprints)
- CLI subcommands `cert info`, `cert rotate-server`
- `serve` startup wiring (ensure CA → ensure leaf)

## Detailed Requirements

1. Files `<data_dir>/ca/{ca.crt,ca.key,server.crt,server.key}` — PEM; dir
   `0700`, keys `0600`. Startup must fail closed with a hard error if either
   private key is group/world-readable or unreadable; the error identifies the
   offending path and preserves the `0700`/`0600` contract.
2. **EnsureCA** (idempotent): ECDSA P-256; subject
   `CN=OpenIntercom Home CA <8 hex chars from crypto/rand>`; validity 10 y;
   `IsCA`, `MaxPathLen=0, MaxPathLenZero=true`; KeyUsage
   `CertSign|CRLSign`; **NameConstraints, critical**:
   `PermittedDNSDomains = ["<mdns_name>.local"] + [hostname if set]`
   (exact-label semantics are fine: entries without a leading dot constrain
   the exact name and subdomains — add both the bare label form we use).
   No IP constraints (unconstrained by omission, RFC 5280 §4.2.1.10).
3. **EnsureLeaf**: desired SANs = DNS: `<mdns_name>.local`, `hostname?`,
   `localhost`; IP: `127.0.0.1` + every non-loopback **private** IPv4
   (RFC 1918) currently assigned (helper `currentPrivateIPv4s()`). Reissue iff:
   file missing, `NotAfter < now+30d`, or SAN set ≠ desired (order-insensitive
   compare). Validity 825 d; EKU `ServerAuth`; ECDSA P-256; signed by the CA.
4. If config changes make the CA's constraints stop covering a desired DNS SAN
   (e.g. user edits `mdns_name` after CA creation): **fail startup** with an
   error explaining the mismatch and the two ways out (revert config, or
   documented CA rotation = delete `ca/` + re-trust all devices).
5. API: `Manager.GetCertificate(*tls.ClientHelloInfo)` serving the current
   leaf (atomic pointer; rotation without restart), `Manager.CAPEM()`,
   `Manager.CADER()`, `Manager.Fingerprint()` (SHA-256 colon-hex),
   `Manager.LeafInfo()` (SANs, NotAfter), `Manager.RotateServer()` (force
   reissue).
6. CLI `cert info`: prints CA subject, fingerprint, CA NotAfter, leaf SANs,
   leaf NotAfter — plain text, stable ordering (used by humans for TOFU
   comparison per §6.6). `cert rotate-server`: calls RotateServer against the
   data dir (hub not required to be running; take the bbolt-independent file
   lock via `ca/.lock` + O_EXCL to avoid racing a live hub, or document
   "stop the hub first" and detect the running hub via the bbolt lock —
   choose the simpler: attempt `store.Open` with 1 s timeout purely as a
   liveness probe and refuse with a clear message if locked).
7. Every generation/rotation appends nothing to the event log from this
   package (no store dependency); `serve` logs `cert.rotated` events at the
   wiring layer (issue 12 exposes them via API).

## Acceptance Criteria

- [ ] Fresh start creates CA+leaf; `openssl verify -CAfile ca.crt server.crt`
      passes; NameConstraints extension present, critical, DNS-only (asserted
      by parsing in tests, not by shelling to openssl).
- [ ] `crypto/x509` verification succeeds for `openintercom.local` and for an
      IP SAN dial; fails for `example.com`.
- [ ] Changing the machine's IP set (simulated by injecting the IP provider)
      triggers exactly one leaf reissue; unchanged set triggers none.
- [ ] Leaf expiring in 29 d reissues; 31 d does not (fake clock).
- [ ] `mdns_name` change after CA creation → startup fails with the actionable
      message.
- [ ] `cert info` output contains the same fingerprint as
      `Manager.Fingerprint()`.
- [ ] Unsafe or unreadable `ca.key` and `server.key` each fail startup closed,
      identify the path, and do not continue to serving.

## Validation

`go test -race ./internal/ca/...` with injected clock + IP provider; a
`tls.Server`/`tls.Client` in-test handshake pinned to the generated CA.
The focused permission cases must cover both private-key paths and both
group/world-readable and unreadable failures.

## Dependencies

03, 04.

## Non-goals

Let's Encrypt mode (v2), CA rotation UX beyond the documented manual
procedure, CRL/OCSP (nothing to revoke — single leaf), serving `/ca.crt`
(issues 08/09).

## Design References

DESIGN.md §6.5, §6.6, §9.4, §9.5; ADR-002;
research/secure-context-lan-https.md.
