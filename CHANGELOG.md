# Changelog

## v0.4.0 — 2026-05-30

Adds typed, accumulated evidence trail for findings (PRD-docket-evidence).

New `evidence` table (M2 migration, append-only) stores typed refs keyed to
finding + run. `docket report --evidence <kind>:<ref>` (repeatable) parses
known prefixes (`recall:`, `journal:`, `pid:`, `provfs:`, `commit:`, `path:`)
into `(kind, ref_val)` rows; unknown prefixes stored as `raw`. Malformed refs
never fail the report.

`docket show <key>` text output gains an *Evidence* section grouped by run;
`--format json` includes `evidence_trail` array. `docket list --format json`
includes `evidence_count` per finding. Migration is idempotent on existing DBs.

All 10 acceptance criteria covered by 10 new integration tests (45 total pass).

## v0.3.0 — 2026-05-30

Adds `docket digest [--format text|json] [--severity <min>]` — a compact
rollup of open/escalated findings for SessionStart banners and health checks.

Text output is a ≤3-line banner (open count, crit, escalated, oldest key).
JSON output is a `wm.health.*`-compatible envelope matching companion-degrade's
`ComponentHealth` shape (component / status / summary / detail).

Status mapping: 0 findings → ok; open-only → ok; any escalated → degraded;
any escalated crit → down. All branches covered by tests.

`--severity warn` excludes info-severity findings from counts and summary.
