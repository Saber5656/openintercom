# 03: CLI skeleton, logging, version

## Summary

Cobra-based CLI with `serve` (stub) and `version`, structured logging via
`log/slog`, and ldflags version injection.

## Context

DESIGN §6.1 fixes the command surface; later issues attach real subsystems to
`serve`. Logging conventions (§6.14) are established here so every module logs
uniformly.

## Scope

- `cmd/openintercom/main.go`, `internal/cli/` (command wiring)
- `internal/version/`, logging bootstrap

## Detailed Requirements

1. Add `github.com/spf13/cobra` (permitted by ADR-004). Root command
   `openintercom` with `--config PATH` persistent flag.
2. `internal/version`: `var (Version = "dev"; Commit = "none"; Date = "unknown")`
   + `String()` → `openintercom <Version> (<Commit>, <Date>)`. Makefile `build`
   passes `-ldflags "-X …"` from `git describe --tags --always` and UTC date.
3. `openintercom version`: prints `version.String()`, exit 0.
4. Logging bootstrap `internal/cli/logging.go`: builds a `*slog.Logger` from
   config (`log_level`, `log_format`; text → `slog.NewTextHandler`, json →
   JSONHandler), sets `slog.SetDefault`. Until issue 04 merges, read the two
   values from env `OPENINTERCOM_LOG_LEVEL/FORMAT` with defaults; switch to
   config struct when 04 lands (coordinate via small interface
   `type LogConfig interface{ LogLevel() string; LogFormat() string }`).
5. `openintercom serve` (stub): initializes logging, logs
   `msg="openintercom starting" version=…`, then blocks on a context cancelled
   by SIGINT/SIGTERM (`signal.NotifyContext`), logs `msg="shutdown complete"`,
   exit 0. All later subsystems will register start/stop hooks here: define
   `type Runner interface{ Run(ctx context.Context) error }` and run a slice of
   Runners with `errgroup`; on first error cancel all and exit non-zero.
6. Unknown subcommand/flag → cobra default error + usage, exit ≠0.

## Acceptance Criteria

- [ ] `./bin/openintercom version` prints injected values (not `dev`) when
      built via `make build`.
- [ ] `serve` starts, logs the start line, exits 0 within 5 s of SIGTERM,
      logging the shutdown line.
- [ ] `OPENINTERCOM_LOG_FORMAT=json serve` emits JSON logs.
- [ ] Unit tests cover version string formatting and Runner-group
      error/cancel propagation (fake runners).

## Validation

`go test -race ./internal/...`; manual: run `serve`, send SIGTERM, inspect
logs in both formats.

## Dependencies

01.

## Non-goals

Config file parsing (04), HTTP servers (08), `cert *` (06),
`admin set-password` (07), `healthcheck` (31).

## Design References

DESIGN.md §6.1, §6.14.
