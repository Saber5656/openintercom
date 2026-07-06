# OpenIntercom documentation index

This directory is the **canonical source of truth** for requirements,
architecture, design and issue planning. GitHub Issues and Pull Requests are
derived artifacts; on conflict, fix these files first.

| Document | Purpose |
|---|---|
| [DESIGN.md](DESIGN.md) | Complete v1 design: product, architecture, hub, PWA, admin, security model, reliability, packaging, testing. Section numbers (§) are stable anchors referenced everywhere else. |
| [ISSUE_PLAN.md](ISSUE_PLAN.md) | v1 completion statement, the 34-issue plan, dependencies, waves, DESIGN coverage map, validation strategy, v2 deferrals, known unknowns. |
| [issues/](issues/) | One implementation-ready draft per issue (`NN-short-title.md`), mirrored to GitHub Issues. |
| [decisions/](decisions/) | ADRs — major architecture decisions and their rationale. |
| [research/](research/) | Research notes that materially shaped the design. |
| `user/` | End-user documentation (created by issues 32/33). |
| `release-checks/` | Per-release manual device-matrix checklists (created by issue 31). |

Reading order for a new contributor: DESIGN §1–§4 → ADR-001…006 →
ISSUE_PLAN → the issue you picked.
