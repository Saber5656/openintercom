# 31: Packaging and release pipeline

## Summary

Everything that turns green `main` into installable artifacts: goreleaser
binaries, multi-arch Docker image on GHCR, hardened systemd unit, the
`healthcheck` subcommand, checksums, and the tag-triggered release
workflow with the manual device-matrix gate.

## Context

DESIGN §11. Target boxes: Raspberry Pi (arm64/armv7), NAS/x86 Linux,
macOS, Windows. Distribution promise: one static binary or one container,
data in one directory.

## Scope

- `.goreleaser.yaml`, `Dockerfile`, `.github/workflows/release.yml`
- `deploy/systemd/openintercom.service`
- CLI `healthcheck` subcommand
- `docs/release-checks/TEMPLATE.md` + release process doc

## Detailed Requirements

1. `healthcheck` subcommand (§6.1): GET `--url` (default
   `https://127.0.0.1:8443/healthz` honoring config if readable),
   `InsecureSkipVerify` **only** for 127.0.0.1 targets (it's a liveness
   probe, but don't normalize skipping verification elsewhere), timeout
   3 s, exit 0/1, no output on success.
2. `.goreleaser.yaml`: builds matrix per §11 (`CGO_ENABLED=0`, ldflags
   version/commit/date); archives tar.gz (zip for windows) containing
   binary + LICENSE + README + `config.example.yaml` +
   `deploy/systemd/openintercom.service`; `checksums.txt` SHA-256;
   changelog from conventional-ish commit grouping (best effort);
   snapshot mode via `make release-snapshot` for local verification.
   Prerequisite in CI: `make build-web` before goreleaser (embedded
   assets must be the release build).
3. `Dockerfile`: multi-stage — node stage builds `web/dist`; Go stage
   builds the binary embedding it; final `FROM gcr.io/distroless/static:
   nonroot`; `USER nonroot`; `VOLUME /data`;
   `ENV OPENINTERCOM_DATA_DIR=/data`; `EXPOSE 8443 8080`;
   `HEALTHCHECK CMD ["/openintercom","healthcheck"]`. Multi-arch
   (amd64/arm64/armv7) via buildx QEMU in the release workflow, pushed to
   `ghcr.io/saber5656/openintercom` with tags `vX.Y.Z` + `latest`.
4. `release.yml`: trigger `push: tags: ['v*']`; jobs: full CI reuse
   (call ci.yml via `workflow_call`) → goreleaser (GitHub release,
   artifacts, checksums) → docker buildx push. Permissions minimal
   (`contents: write` release job; `packages: write` docker job); all
   actions SHA-pinned. **No auto-release without a tag**; tagging policy
   documented: tag only after the §12.4 checklist file for that version
   is committed to `docs/release-checks/` (the workflow greps for it and
   fails the release if absent — mechanical gate).
5. systemd unit: `User=openintercom`, `ExecStart=/usr/local/bin/
   openintercom serve --config /etc/openintercom/config.yaml`,
   `Restart=on-failure`, hardening per §11 (`NoNewPrivileges`,
   `ProtectSystem=strict`, `ReadWritePaths=/var/lib/openintercom`,
   `PrivateTmp`, `ProtectHome=read-only`, `CapabilityBoundingSet=`,
   `RestrictAddressFamilies=AF_INET AF_INET6 AF_NETLINK`); unit file
   comments explain each line briefly (teachable OSS).
6. `docs/release-checks/TEMPLATE.md`: the §12.4 checklist as markdown
   checkboxes + device/OS/browser table + result fields.
7. Version bump/tag process documented in CONTRIBUTING (release section):
   snapshot → checklist on devices → commit checklist → tag → workflow.

## Acceptance Criteria

- [ ] `make release-snapshot` produces runnable binaries for all six
      targets (spot-run linux/amd64 + host platform; `file` check the
      rest) with correct `version` output.
- [ ] `docker run -p 8443:8443 -v oi:/data ghcr.io/…:snapshot` (local
      build) serves the app; container healthcheck goes healthy; runs as
      nonroot (`docker inspect` UID ≠ 0).
- [ ] armv7 image boots on a real RPi (or QEMU) to the ready log line —
      recorded in PR.
- [ ] systemd unit passes `systemd-analyze verify` and runs on a Debian
      VM with the documented install steps (hub survives reboot).
- [ ] Tag `v0.0.1-rc1` on a scratch branch executes the full release
      workflow into a draft/prerelease with all artifacts + checksums,
      and **fails** when the release-checks file is absent (both
      demonstrated).
- [ ] Checksums verify (`sha256sum -c`).

## Validation

Snapshot + scratch-tag dry runs; RPi/QEMU boot; checklist-gate negative
test.

## Dependencies

02, 03, 19 (embedded UI must exist for meaningful artifacts).

## Non-goals

Homebrew/apt/AUR packaging (v2, community), auto-update mechanism
(reload-based UI updates only), signing beyond checksums (cosign = v2
candidate, noted in SECURITY.md).

## Design References

DESIGN.md §6.1 (healthcheck), §11, §12.4; ADR-004, ADR-005.
