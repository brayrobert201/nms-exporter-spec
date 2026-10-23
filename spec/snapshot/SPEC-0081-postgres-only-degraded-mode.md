---
id: SPEC-0081
title: 'Postgres only: behaviour when the store is down'
description: 'The map''s history comes only from Postgres; with it unconfigured or unreachable the history APIs answer 503 with a JSON reason, the pages show a Database down banner, and save metrics carry on.'
type: snapshot
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: '2026-10-23'
status: current
depends_on: [SPEC-0080]
supersedes: []
superseded_by: []
related_specs: [SPEC-0090, SPEC-0083, SPEC-0099, SPEC-0002, SPEC-0061, SPEC-0068, SPEC-0069]
screenshots: [screenshots/database-down.png]
---

# SPEC-0081: Postgres only: behaviour when the store is down

## Summary

The exporter has two halves with different failure rules. The save half (metrics, `/json`,
wall screens, Grafana; SPEC 0001–0019) reads only local files and must keep working through any
outage. The history half (the map's trail, elevation, finds, sectors and star systems; SPEC
0060–0079) reads only from Postgres (SPEC-0080). There is deliberately **no file fallback**: if
the database is unconfigured or unreachable, the history endpoints fail loudly with HTTP 503 and
a machine-readable reason, the pages show a "Database down" banner, and a gauge reports it. The
owner chose this over a silent fallback to the hook's files (SPEC-0090); a rebuild must not
"fix" it into one.

## Inputs and outputs

| Endpoint | Store needed | When the store is unavailable |
|---|---|---|
| `GET /metrics` | no | 200, plus `nms_db_up 0` |
| `GET /json`, `/diff`, `/raw/bases`, `/healthz`, `/` | no | unaffected |
| `GET /api/map` | partly | 200; the save's planets and bases still listed; planets known only from the hook are missing; `errors["nms-db"]` explains |
| `GET /api/elevation`, `/api/finds`, `/api/sectors`, `/api/systems` | yes | 503 `{"error": "...", "db": false}` |
| `GET /api/live` | partly | 200; position, heading, weather and the save's nearby objects; `db: false`, `db_error`; no trail, names or finds |
| `POST /api/finds/class` | yes | 503 `{"ok": false, "error": "nms-db unreachable: ..."}` |
| `GET /map`, `/systems` (HTML) | no | page loads; banner shown once an API call reports `db: false` |

The 503 body shape is fixed:

```json
{"error": "nms-db not configured: no PGHOST (... writes .../db.env)", "db": false}
{"error": "nms-db unreachable: connection timeout expired", "db": false}
```

`db` is always the literal `false` in an error body; clients test `got.db === false`, never
the status alone, because the same check runs on successful bodies (which have no `db` key).

## Requirements

### SPEC-0081-REQ-001: History route list

The exporter shall treat exactly these paths as history routes: `/api/live`, `/api/elevation`,
`/api/finds`, `/api/sectors`, `/api/systems`. (`/api/live` is on the list because the response
includes the recent trail and nearby finds, even though the current position itself comes from
`live.json`.)

### SPEC-0081-REQ-002: Unconfigured store is checked first

If a history route is requested while the store is not configured (`PGHOST` empty), the exporter
shall answer 503 with `{"error": <reason>, "db": false}` before doing any other work, including
refreshing the save. The reason shall name the missing variable and where it would come from.
This check must come before any database call: without it, libpq with no `PGHOST` silently
tries a local Unix socket and the user sees a confusing socket error instead of "not
configured" (this happened on the first run; see lessons).

### SPEC-0081-REQ-003: Any database error becomes 503

If any database driver error escapes a GET handler, the exporter shall answer 503 with
`{"error": "nms-db unreachable: <driver message>", "db": false}`. Other exceptions remain 500
with a plain-text body.

### SPEC-0081-REQ-004: The map endpoint degrades, it does not fail

When `/api/map` is requested and the store is unconfigured or raises a driver error while adding
hook-only planets or refreshing systems, the exporter shall still answer 200 with the planets and
bases built from the save, and set `errors["nms-db"] = "nms-db unreachable: <reason>"`. When
unconfigured it shall not attempt a connection at all (it raises the "not configured" reason
itself).

### SPEC-0081-REQ-005: Recording a hotspot label

If `POST /api/finds/class` hits a database error, the exporter shall answer 503 with
`{"ok": false, "error": "nms-db unreachable: <reason>"}` and shall not keep the label anywhere
else.

### SPEC-0081-REQ-006: Gauge

The exporter shall append to every `/metrics` response:

```
# HELP nms_db_up 1 when nms-db (the map's history store) is reachable and ingesting
# TYPE nms_db_up gauge
nms_db_up 1
```

with value 0 when the store is unconfigured or the ingest loop's last pass (or the migration)
failed (`ingest.error` non-empty), else 1. The gauge reflects the writer's view; a read-only
second instance that cannot take the writer lock reports 1.

### SPEC-0081-REQ-007: Save half never blocked

The save-side endpoints (`/metrics` apart from the extra gauge, `/json`, `/diff`, `/raw/bases`,
`/healthz`) shall not open a database connection, and their latency shall not depend on the
database. The ingest thread runs separately, and its failures only change `ingest.error`.

### SPEC-0081-REQ-008: Map banner

When any `/api/*` response the map page receives has `db === false`, the map shall show a
banner: a critical-status pill reading "⚠ Database down" followed by
"History, finds and sectors can't load: <error>". When a later response is fine, the banner
shall disappear. A 503 from `POST /api/finds/class` shall feed the same check. The banner follows
the map's status rule: never colour alone, always glyph plus words (SPEC 0060–0079).

### SPEC-0081-REQ-009: Systems page banner

When `/api/systems` returns `db === false`, the systems page shall show, in place of its known-glyphs line,
"⚠ Database down: the systems list is in nms-db, which can't be reached (<error>)." and render
no system list.

### SPEC-0081-REQ-010: No stale history

The exporter shall not serve history from the hook's day files or from any in-memory cache that
outlived a database error as if it were current. (In-memory reader caches are refreshed by
`tail()` before each response; a failing `tail()` fails the response.)

## Degraded behaviour

This whole spec is the degraded behaviour. Summary of what a user sees:

| Store state | Wall screens / Grafana | `/map` | `/systems` | `/metrics` |
|---|---|---|---|---|
| configured, up | normal | normal | normal | `nms_db_up 1` |
| unconfigured | normal | save planets and bases; banner "Database down ... nms-db not configured" | banner | `nms_db_up 0` |
| configured, down | normal | save planets and bases; banner with the driver message; live position still drawn | banner | `nms_db_up 0` |
| down, then back | normal | banner clears on the next successful poll; nothing lost (SPEC-0080 offsets) | recovers | back to 1 within one ingest pass |

`/api/live` is the exception to "Postgres only" because the position comes from a file, not the
database: `live.build` guards every database read, and the first failure sets `db: false` and
`db_error` while the rest of the response is still built. (Until 2026-10-23 the route was on the
503 list and the player vanished with the database; that was a cost of the first cut, not a
choice.)

## Verification

1. Start the exporter with no `PGHOST`:
   - `curl -s -o /dev/null -w '%{http_code}' localhost:9877/api/finds` prints `503`;
   - `curl -s localhost:9877/api/finds` prints `{"error": "nms-db not configured: ...", "db": false}`;
   - `curl -s localhost:9877/metrics | grep '^nms_db_up'` prints `nms_db_up 0`;
   - `curl -s localhost:9877/json` is the normal save snapshot;
   - `curl -s localhost:9877/api/map | python3 -c 'import json,sys; print(json.load(sys.stdin)["errors"])'`
     shows the `nms-db` entry and the save's planets are present;
   - the log does **not** mention a Unix socket.
2. With a configured store, block it (stop the database, or firewall the port): within one
   2 s pass `nms_db_up` is 0, `/api/finds` is 503 with a driver message, and `/map` shows the
   red "Database down" pill. Unblock: the banner clears without reloading the page.
3. `POST /api/finds/class` with `{"key":"...","class":"S"}` while blocked returns 503 and
   `{"ok": false, ...}`; after unblocking, the label is not there (nothing was queued).

## Lessons

See `spec/lessons/SPEC-0099-storage-deploy-lessons.md`: check "configured" before touching the
driver, or an unset `PGHOST` becomes a Unix-socket error.
