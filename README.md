# docket

A structured ledger for standing findings. A finding reported twice is the same finding.

`docket` is a small SQLite-backed CLI that deduplicates findings by a stable key, tracks
first/last-seen timestamps, maintains a consecutive-run streak, and accumulates a typed
**evidence trail** — so every run that observes a finding contributes its proof, and
`docket show` renders the full chronological trail.

## Install

```sh
cargo install --path .
# binary lands at ~/.cargo/bin/docket
# or copy target/release/docket to ~/.local/bin/
```

## Database location

`$XDG_DATA_HOME/docket/docket.db` (defaults to `~/.local/share/docket/docket.db`).

The directory is created automatically on first use.

## Schema

### `findings` table

| column              | type    | description                                                      |
|---------------------|---------|------------------------------------------------------------------|
| `key`               | TEXT PK | stable slug — `agorabus-stale-binary`                           |
| `title`             | TEXT    | human one-liner (latest report wins)                            |
| `severity`          | TEXT    | `info` / `warn` (default) / `crit`                              |
| `status`            | TEXT    | `open` / `escalated` / `resolved`                               |
| `first_seen`        | TEXT    | RFC3339 timestamp of first report                               |
| `last_seen`         | TEXT    | RFC3339 timestamp of most recent report                         |
| `first_run`         | TEXT    | run-id of the first report                                      |
| `last_run`          | TEXT    | run-id of the most recent report                                |
| `runs_seen`         | INTEGER | count of distinct run-ids that reported this finding            |
| `consecutive_runs`  | INTEGER | current streak (distinct sequential run-ids)                    |
| `report_count`      | INTEGER | total raw report calls                                          |
| `resolved_at`       | TEXT    | RFC3339 timestamp of resolution (null if open)                  |
| `resolve_reason`    | TEXT    | reason string (null if not provided)                            |
| `escalated_at`      | TEXT    | RFC3339 timestamp of escalation (null if not escalated)         |
| `escalation_reason` | TEXT    | escalation reason (null if not escalated)                       |

### `evidence` table (M2 — append-only)

Each `--evidence` ref is stored as a typed row, keyed by `findings.key` and the reporting `run_id`.
Evidence rows are never overwritten — every new report appends to the trail.

| column    | type    | description                                                      |
|-----------|---------|------------------------------------------------------------------|
| `id`      | INTEGER | auto-increment primary key (insertion order)                    |
| `key`     | TEXT    | FK → `findings.key`                                             |
| `run_id`  | TEXT    | run identifier of the reporter                                  |
| `kind`    | TEXT    | evidence kind (see table below)                                 |
| `ref_val` | TEXT    | the reference value (everything after the `<kind>:` prefix)    |
| `note`    | TEXT    | reserved (always null for now)                                  |
| `seen_at` | TEXT    | RFC3339 timestamp when the row was inserted                     |

### Evidence kinds

| prefix     | kind      | meaning                            | example                                |
|------------|-----------|------------------------------------|----------------------------------------|
| `recall:`  | `recall`  | recall memory ULID                 | `recall:01KSRV7R4FERPP40HQGV5RGZNT`  |
| `journal:` | `journal` | journal date + optional `#line`    | `journal:2026-05-28#7`                |
| `pid:`     | `pid`     | process id observed                | `pid:2138939`                          |
| `provfs:`  | `provfs`  | `user.prov.ts` epoch / xattr       | `provfs:1780026726`                    |
| `commit:`  | `commit`  | git SHA                            | `commit:02350fb`                       |
| `path:`    | `path`    | filesystem path                    | `path:/home/jsy/.local/bin/agorabus`  |
| (other)    | `raw`     | unknown prefix — stored as-is      | `somethingweird`                       |

Parsing is always **lenient**: malformed refs (e.g. invalid ULIDs) are stored as-given and never
cause a nonzero exit.

WAL mode is enabled; `BEGIN IMMEDIATE` transactions guard concurrent writers.

## Run model

A **run** is a caller-supplied opaque string (e.g. `2026-05-29.1`) passed via `--run`.

- Reporting the same key **twice in the same run-id**: `report_count` increments, `runs_seen`
  and `consecutive_runs` do not. Idempotent for streak purposes.
- Reporting the same key in a **new run-id**: both `runs_seen` and `consecutive_runs` increment.
- Reporting a **resolved** finding: it is reopened (`status=open`, streak reset to 1,
  `resolved_at` cleared).

## Commands

### `report`

Upsert a finding. `--evidence` is repeatable; pass it once per ref.

```sh
docket report \
  --run <id> \
  --key <slug> \
  --title <text> \
  [--severity info|warn|crit] \
  [--evidence <kind>:<ref>] \
  [--evidence <kind>:<ref>] ...
```

- Creates the finding if new (`status=open`, streak=1, `runs_seen=1`, `report_count=1`).
- Bumps an existing open finding with the same run-id: increments `report_count` only.
- Bumps an existing open finding with a new run-id: increments `runs_seen`, `consecutive_runs`,
  `report_count`, updates `last_seen`/`last_run`.
- Reopens a resolved finding: resets `consecutive_runs=1`, clears `resolved_at`/`resolve_reason`.
- Each `--evidence` ref appends one row to the `evidence` table tagged with the `run_id`.

### `list`

List findings.

```sh
docket list [--open|--resolved|--escalated|--all] [--format text|json] [--severity info|warn|crit]
```

Default: `--open --format text`. `--severity` is a minimum filter (inclusive).

JSON output includes `evidence_count` (aggregate) per finding but not the full trail.

### `show`

Show a single finding's full record, including its complete evidence trail.

```sh
docket show <key> [--format text|json]
```

- Text output includes an *Evidence* section grouped by run, each line `[<run_id>] <kind>: <ref>`.
- JSON output includes an `evidence_trail` array of typed `EvidenceRow` objects.
- Exits nonzero if the key is unknown.

### `resolve`

Mark a finding as resolved.

```sh
docket resolve <key> [--reason <text>]
```

Idempotent. Exits nonzero if the key is unknown.

### `sweep`

Auto-resolve stale open/escalated findings not seen in recent runs.

```sh
docket sweep --run <current-run-id> [--stale-after <N>]
```

Default `stale-after`: 3 (overridden by `DOCKET_STALE_AFTER` env var).

## Worked example — multi-run evidence trail

The real value of typed evidence is that a single finding accumulates proof across multiple
self-review runs. Here is the `agorabus-stale-binary` case that motivated this feature:

```sh
# Run 18 (ULID 01KSRV7R4FERPP40HQGV5RGZNT) sees the binary and a running pid
docket report \
  --run 01KSRV7R4FERPP40HQGV5RGZNT \
  --key agorabus-stale-binary \
  --title "agorabus binary is stale (running a deleted inode)" \
  --severity crit \
  --evidence recall:01KSRV7R4FERPP40HQGV5RGZNT \
  --evidence pid:2138939 \
  --evidence provfs:1780026726

# Run 19 (ULID 01KSS21WFN5H6V42JF723Z8K2J) — still there, plus a journal entry
docket report \
  --run 01KSS21WFN5H6V42JF723Z8K2J \
  --key agorabus-stale-binary \
  --title "agorabus binary is stale (running a deleted inode)" \
  --evidence recall:01KSS21WFN5H6V42JF723Z8K2J \
  --evidence journal:2026-05-29#7 \
  --evidence path:/proc/2138939/exe

# Show the full trail
docket show agorabus-stale-binary
# [crit] agorabus-stale-binary (crit)
#   title: agorabus binary is stale (running a deleted inode)
#   ...  runs_seen: 2  consecutive_runs: 2  report_count: 2
#   Evidence:
#     [01KSRV7R4FERPP40HQGV5RGZNT] recall: 01KSRV7R4FERPP40HQGV5RGZNT
#     [01KSRV7R4FERPP40HQGV5RGZNT] pid: 2138939
#     [01KSRV7R4FERPP40HQGV5RGZNT] provfs: 1780026726
#     [01KSS21WFN5H6V42JF723Z8K2J] recall: 01KSS21WFN5H6V42JF723Z8K2J
#     [01KSS21WFN5H6V42JF723Z8K2J] journal: 2026-05-29#7
#     [01KSS21WFN5H6V42JF723Z8K2J] path: /proc/2138939/exe

# Machine-readable evidence trail
docket show agorabus-stale-binary --format json | jq '.evidence_trail[] | "\(.run_id) \(.kind):\(.ref_val)"'

# List with evidence counts
docket list --format json | jq '.[] | "\(.key): \(.evidence_count) evidence refs"'
```

## License

MIT OR Apache-2.0
