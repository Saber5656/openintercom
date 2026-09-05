# 04: Configuration loading and validation

## Summary

`internal/config`: YAML file + env + flag precedence, strict validation, all
errors reported at once, per DESIGN §6.2.

## Context

Every subsystem consumes the immutable config struct. §6.2 is the complete key
table; deployment concerns only (runtime settings live in the store, §6.3).

## Scope

- `internal/config/` (`Load`, struct, validation), `config.example.yaml`

## Detailed Requirements

1. Struct fields exactly the §6.2 table: `ListenHTTPS`, `ListenHTTP`,
   `Hostname`, `MDNSName`, `AdvertiseMDNS`, `DataDir`, `LogLevel`, `LogFormat`.
2. `Load(opts)` resolution: explicit `--config` path > `$OPENINTERCOM_CONFIG` >
   `./config.yaml` > defaults. A missing file is fine **unless** the path was
   explicit (then: error). Use `gopkg.in/yaml.v3` with `KnownFields(true)` —
   unknown keys are a hard error listing each offending key.
3. Env overrides `OPENINTERCOM_<UPPER_SNAKE>` applied after file; booleans
   parse `true/false/1/0`; empty string env counts as set (allows disabling
   `listen_http`).
4. Validation (collect all failures into one multi-error):
   host:port syntax for both listeners (empty allowed only for `listen_http`);
   `hostname`/`mdns_name` RFC 1123 label(s), lowercase; `log_level` ∈
   {debug,info,warn,error}; `log_format` ∈ {text,json}; `data_dir` non-empty.
5. `EnsureDataDir(cfg)`: create `0700` if missing; error if world-writable or
   not writable (probe file).
6. `config.example.yaml`: every key, default value, one-line comment each —
   kept in sync by a unit test that parses it and asserts equality with
   defaults.
7. Implements the `LogConfig` interface from issue 03.

## Acceptance Criteria

- [ ] Defaults load with no file present; explicit missing path errors.
- [ ] Unknown key `listen_htps:` produces an error naming that key.
- [ ] `OPENINTERCOM_LISTEN_HTTP=""` disables the setup listener (visible to
      callers as empty string).
- [ ] Multi-error output lists *all* invalid fields in one run.
- [ ] Example file drift breaks a test.

## Validation

Table-driven `go test -race ./internal/config/...` covering: precedence
(file<env<flag), each validation rule negative + positive, boolean env
parsing, data-dir permission failure (use `t.TempDir` + chmod).

## Dependencies

03.

## Non-goals

Hot reload; store-backed runtime settings (§6.3, issue 05); TLS paths (the CA
subsystem owns its file locations under `data_dir`, issue 06).

## Design References

DESIGN.md §6.2, §6.14.
