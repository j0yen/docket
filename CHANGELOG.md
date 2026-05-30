# Changelog

## v0.3.0 — 2026-05-30

Adds `docket digest [--format text|json] [--severity <min>]` — a compact
rollup of open/escalated findings for SessionStart banners and health checks.

Text output is a ≤3-line banner (open count, crit, escalated, oldest key).
JSON output is a `wm.health.*`-compatible envelope matching companion-degrade's
`ComponentHealth` shape (component / status / summary / detail).

Status mapping: 0 findings → ok; open-only → ok; any escalated → degraded;
any escalated crit → down. All branches covered by tests.

`--severity warn` excludes info-severity findings from counts and summary.
