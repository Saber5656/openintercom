# 01: Repository scaffold and tooling baseline

## Summary

Create the Go module, directory skeleton, Makefile, license and contributor
baseline files so every later issue lands into a fixed structure.

## Context

The repository currently contains only a README. DESIGN.md §5 fixes the layout
and module path; this issue materializes it without implementing any product
behavior.

## Scope

- `go.mod` / directory tree / placeholder packages
- `Makefile`, `.gitignore`, `.editorconfig`
- `LICENSE` (MIT), `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, GitHub templates

## Detailed Requirements

1. `go mod init github.com/Saber5656/openintercom`; `go 1.24` directive.
2. Create directories exactly as DESIGN §5: `cmd/openintercom/`,
   `internal/{config,store,ca,auth,httpserver,api,ws,signal,mdns,events,webassets,version}/`,
   `web/` (empty placeholder with `.gitkeep`), `deploy/systemd/`,
   `.github/workflows/` (empty), keeping existing `docs/`.
   Each `internal/*` package gets a `doc.go` with package comment quoting its
   DESIGN section number.
3. `cmd/openintercom/main.go`: minimal `main` that prints
   `openintercom (dev build)` and exits 0 (replaced by issue 03).
4. `Makefile` targets (all must run green now, even as partial no-ops):
   `build` (build-web if `web/package.json` exists, then `go build -o bin/openintercom ./cmd/openintercom`),
   `build-go`, `build-web` (no-op with notice until issue 19), `test`
   (`go test -race ./...`), `test-web` (no-op until 19), `lint`
   (`golangci-lint run` if installed, else instruct), `fmt` (`gofmt -l -w .`),
   `clean`. Document each target with a `## help` comment convention and a
   `help` target.
5. `.gitignore`: `bin/`, `data/`, `web/node_modules/`, `web/dist/`,
   `*.local.yaml`, OS junk. `.editorconfig`: tabs for Go, 2-space for
   ts/json/yaml/md, LF, final newline.
6. `LICENSE`: MIT, copyright `2026 OpenIntercom contributors`.
7. `CONTRIBUTING.md`: dev prerequisites (Go ≥1.24, Node LTS), `make` targets
   table, branch/PR convention (PRs to `main`, no direct push), link to
   DESIGN.md and ISSUE_PLAN.md as canonical docs, note that new dependencies
   require an ADR-004 update.
8. `CODE_OF_CONDUCT.md`: Contributor Covenant v2.1 verbatim with contact
   placeholder `security contact in SECURITY.md`.
9. `.github/ISSUE_TEMPLATE/bug_report.md`, `feature_request.md` (minimal), and
   `.github/pull_request_template.md` (checklist: tests, docs, ADR-004 dep
   rule).

## Acceptance Criteria

- [ ] `make build && ./bin/openintercom` prints the placeholder line, exit 0.
- [ ] `make test`, `make fmt`, `make help` all succeed on a clean checkout.
- [ ] `go vet ./...` clean; tree matches DESIGN §5 exactly (no extra dirs).
- [ ] All baseline files above exist with the specified content characteristics.

## Validation

`make build test fmt` locally; `git status` clean after `make build`
(artifacts ignored). Reviewer diffs tree against DESIGN §5.

## Dependencies

None.

## Non-goals

CI workflows (02), real CLI (03), frontend project (19), any product logic.

## Design References

DESIGN.md §5; ADR-004; ISSUE_PLAN.md wave 0.
