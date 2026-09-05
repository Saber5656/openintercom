# 02: CI pipeline: lint, test, build, security scanning

## Summary

GitHub Actions CI for every PR and `main` push: Go lint/test/build, frontend
placeholders, govulncheck, CodeQL, Dependabot — hardened per DESIGN §9.10.

## Context

CI must exist before feature waves so every subsequent issue merges through
gates (DESIGN §12.2). Frontend jobs activate automatically once issue 19 lands.

## Scope

- `.github/workflows/ci.yml`, `.github/workflows/codeql.yml`
- `.github/dependabot.yml`, `.golangci.yml`

## Detailed Requirements

1. `ci.yml` triggers: `pull_request`, `push` to `main`. Top-level
   `permissions: contents: read`. `concurrency` group per ref with
   `cancel-in-progress: true`. **Every action pinned to a full commit SHA**
   with a `# vX.Y.Z` comment.
2. Jobs:
   - `go-lint`: `golangci-lint` (version pinned in the workflow) using repo
     `.golangci.yml`.
   - `go-test`: `go test -race -count=1 ./...` with Go version from `go.mod`;
     cache modules.
   - `govulncheck`: `golang.org/x/vuln/cmd/govulncheck@<pinned>` over `./...`.
   - `web`: runs only `if: hashFiles('web/package.json') != ''` — steps:
     `npm ci`, `npm run lint`, `npm run typecheck`, `npm test`,
     `npm run build` (issue 19 defines the scripts; job must be green-skip
     until then).
   - `build`: `make build` (proves embed + binary link).
3. `.golangci.yml`: enable at least `govet, staticcheck, errcheck, gosec,
   revive, gofmt, misspell`; exclude generated/embedded dirs; treat findings
   as failures (no `--issues-exit-code=0`).
4. `codeql.yml`: languages `go`, `javascript-typescript`; on PR, push to
   `main`, and weekly schedule; default queries; pinned action SHAs;
   `permissions: security-events: write, contents: read`.
5. `dependabot.yml`: ecosystems `gomod`, `npm` (dir `/web`),
   `github-actions`; weekly; grouped minor/patch updates.
6. README badge for CI status (single badge, `main`).

## Acceptance Criteria

- [ ] A PR touching a `.go` file runs go-lint/go-test/govulncheck/build and all
      pass on the scaffold.
- [ ] `web` job shows as skipped (not failed) while `web/package.json` is absent.
- [ ] `gh api repos/{owner}/{repo}/actions/workflows` lists ci and codeql as
      active; CodeQL completes on `main`.
- [ ] Zero actions referenced by tag only; all SHA-pinned.
- [ ] Dependabot config accepted by GitHub (Insights → Dependency graph shows it).

## Validation

Open a scratch PR with a trivial Go change and a deliberate lint error; verify
red; fix; verify green. Confirm concurrency cancels superseded runs.

## Dependencies

01.

## Non-goals

E2E job wiring (30), release workflow and Docker publishing (31), repository
rulesets/settings (owner-managed, see issue 34 checklist).

## Design References

DESIGN.md §9.10, §12.2; ADR-004 dependency policy.
