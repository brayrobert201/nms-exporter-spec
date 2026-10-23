---
id: SPEC-0080
title: 'History store and ingest'
description: 'The Postgres schema for everything not in the save, and the 2 s ingest that copies the live hook''s JSONL day files into it exactly once.'
type: snapshot
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: []
supersedes: []
superseded_by: []
related_specs: [SPEC-0081, SPEC-0082, SPEC-0083, SPEC-0090, SPEC-0099, SPEC-0042, SPEC-0045, SPEC-0046, SPEC-0048, SPEC-0063, SPEC-0068]
screenshots: []
---

# SPEC-0080: History store and ingest

## Summary

The live hook (SPEC 0040–0059) runs inside the game and appends one JSON object per line to
day files: `trail-YYYYMMDD.jsonl`, `visor-YYYYMMDD.jsonl`, `systems-YYYYMMDD.jsonl` and
`weather-YYYYMMDD.jsonl`, all in one directory inside the game's Proton prefix (the "live
directory", `NMS_LIVE_DIR`). The exporter copies every new complete line of those files into a
single Postgres table, `event`, exactly once, on a background thread every 2 seconds. From then
on every history the map shows (trail, elevation, finds, sectors, star systems; SPEC 0060–0079)
is read back from Postgres, never from the files. Two small key/value tables hold what the
player records on the map (hotspot classes and resources) and per-planet facts learned from the
hook; both were JSON files before and are imported from them once.

The hook's own files stay as they are: they are the raw output and the store's input. The hook
also writes `live.json`, the 4 Hz current-position file, which is state, not history, and is not
ingested (see `docs/data-formats.md`).

## Inputs and outputs

| Direction | What | Format |
|---|---|---|
| in | `<live dir>/{trail,visor,systems,weather}-YYYYMMDD.jsonl` | one JSON object per line, each with a numeric `t` (unix seconds); `docs/data-formats.md` |
| in | `<live dir>/planets.json`, `<live dir>/find_labels.json` | legacy JSON objects, read once at first migration if present |
| in | libpq environment: `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`, `PGSSLMODE` | from the service's env file (SPEC-0082, SPEC-0084) |
| out | Postgres tables `schema_version`, `event`, `ingest_file`, `find_label`, `planet_facts` | DDL below |
| out | Postgres views `trail`, `marker`, `chat`, `star_system`, `storm` | DDL below, for ad-hoc SQL |
| out | `ingest.error`, `ingest.last_ok` | in-process status read by `/metrics` and the 503 handler (SPEC-0081) |

## Schema (normative)

This DDL is the spec. It is applied idempotently on every start (`IF NOT EXISTS` /
`CREATE OR REPLACE`), inside one transaction that first takes `pg_advisory_xact_lock(4207)`, so
two exporters starting together cannot race the DDL. `SCHEMA_VERSION` is 1; a row is inserted
into `schema_version` when `max(version)` is lower.

```sql
CREATE TABLE IF NOT EXISTS schema_version (version int NOT NULL);
CREATE TABLE IF NOT EXISTS event (
    id      bigserial PRIMARY KEY,
    stream  text NOT NULL,             -- trail | visor | systems | weather
    t       double precision NOT NULL, -- unix seconds, as the hook wrote it
    address bigint,                    -- trail: the planet/system (packed like the save's GalacticAddress)
    data    jsonb NOT NULL             -- the hook's line, unchanged
);
CREATE INDEX IF NOT EXISTS event_address ON event (address, t) WHERE address IS NOT NULL;
CREATE INDEX IF NOT EXISTS event_stream_id ON event (stream, id);
CREATE INDEX IF NOT EXISTS event_stream_t ON event (stream, t);
CREATE TABLE IF NOT EXISTS ingest_file (
    path    text PRIMARY KEY,          -- the day file's basename, e.g. trail-20261023.jsonl
    pos     bigint NOT NULL,           -- byte offset just past the last ingested newline
    updated timestamptz NOT NULL DEFAULT now()
);
-- what the player recorded on the map (was <live dir>/find_labels.json)
CREATE TABLE IF NOT EXISTS find_label (
    key     text PRIMARY KEY,
    label   jsonb NOT NULL
);
-- each visited planet's resources, biome, weather, colours (was <live dir>/planets.json)
CREATE TABLE IF NOT EXISTS planet_facts (
    address bigint PRIMARY KEY,
    facts   jsonb NOT NULL,
    updated timestamptz NOT NULL DEFAULT now()
);

-- typed views for queries; positions are metres in the planet's frame (centre = 0, +Y = north)
CREATE OR REPLACE VIEW trail AS
SELECT id, to_timestamp(t) AS at, t, address, (address >> 52) & 15 AS planet,
       data->>'location' AS location,
       (data->'rel'->>0)::float8 AS x, (data->'rel'->>1)::float8 AS y, (data->'rel'->>2)::float8 AS z,
       (data->>'distance_from_planet')::float8 AS altitude, (data->>'sealevel')::float8 AS sealevel
FROM event WHERE stream = 'trail';
CREATE OR REPLACE VIEW marker AS
SELECT id, to_timestamp(t) AS at, t, data->>'type' AS type, data->>'name' AS name,
       (data->'rel'->>0)::float8 AS x, (data->'rel'->>1)::float8 AS y, (data->'rel'->>2)::float8 AS z
FROM event WHERE stream = 'visor' AND data->>'event' = 'marker_add';
CREATE OR REPLACE VIEW chat AS
SELECT id, to_timestamp(t) AS at, t, data->>'event' AS command, data->>'class' AS class,
       data->>'resource' AS resource, data->>'name' AS name,
       (data->'rel'->>0)::float8 AS x, (data->'rel'->>1)::float8 AS y, (data->'rel'->>2)::float8 AS z
FROM event WHERE stream = 'visor' AND data->>'event' IN ('label', 'mark');
CREATE OR REPLACE VIEW star_system AS
SELECT id, to_timestamp(t) AS at, data FROM event WHERE stream = 'systems' AND data->>'event' = 'system';
CREATE OR REPLACE VIEW storm AS
SELECT id, to_timestamp(t) AS at, data FROM event WHERE stream = 'weather';
```

Design notes, so a rebuild does not "improve" them away:

- **One table, `jsonb` payload.** The hook's line is stored unchanged. The hook is exploratory
  and adds fields between versions; a typed table per stream would need a migration every time.
  The views give typed columns where queries need them.
- **`address` is filled only for trail rows.** Visor and systems rows carry no planet; readers
  place them on the planet the trail says the player was on at that `t` (SPEC 0060–0079).
- **`id` is the read cursor.** Readers page by `id`, not `t`, so rows inserted late (a file
  ingested after a restart) are still seen once.

## Requirements

### SPEC-0080-REQ-001: Connection from libpq variables

The exporter shall connect with an empty connection string so libpq reads `PGHOST`, `PGPORT`,
`PGDATABASE`, `PGUSER`, `PGPASSWORD` and `PGSSLMODE` from the environment, with autocommit on,
`connect_timeout=5` and `application_name=nms-exporter`. The store shall count as configured if
and only if `PGHOST` is set and non-empty.

### SPEC-0080-REQ-002: Connection per thread, reconnect when broken

Each thread shall hold its own connection (thread-local). When a thread's connection is
missing, closed or broken, the next call on that thread shall open a new one. (Every HTTP
handler thread therefore has its own connection; at one user this is fine, and a pool is the
upgrade if it ever is not.)

### SPEC-0080-REQ-003: Migration at start, retried

When the exporter starts and the store is configured, the exporter shall run the migration
(schema above, under `pg_advisory_xact_lock(4207)`) on a background thread, retrying every 30
seconds until it succeeds, recording each failure as `ingest.error`, and only then start the
ingest loop. The HTTP server shall start without waiting for it.

### SPEC-0080-REQ-004: One-time import of the legacy JSON files

While migrating, if `planet_facts` has no rows, the exporter shall read `<live dir>/planets.json`
(an object of decimal address string -> facts object) and insert each entry with
`ON CONFLICT DO NOTHING`; likewise if `find_label` has no rows, `<live dir>/find_labels.json`
(an object of find key -> label object). A missing or unparsable file shall be skipped silently.
The files are never written again.

### SPEC-0080-REQ-005: Ingest period

The ingest loop shall run one pass, then sleep 2.0 seconds, forever.

### SPEC-0080-REQ-006: Single writer

At the start of each pass, if this connection does not already hold it, the exporter shall try
`pg_try_advisory_lock(4208)` (a session lock). If the lock is held elsewhere, the pass shall
write nothing and return 0. Because it is a session lock, it is released automatically when the
writer's connection dies, and the next reconnecting exporter takes over. Purpose: a second copy
of the exporter (a test instance on another port, SPEC-0082) reads the same database but never
writes.

### SPEC-0080-REQ-007: Files and order

Each pass shall visit the streams in the order `trail`, `visor`, `systems`, `weather`, and within
each stream every file matching `<stream>-*.jsonl` in the live directory in sorted (that is, date)
order. Offsets shall be loaded once per pass from `ingest_file` keyed by basename.

### SPEC-0080-REQ-008: Complete lines only

For each file whose size exceeds its stored offset, the exporter shall read from the offset, at
most 64 MiB, and keep only up to and including the last `\n`. A half-written final line shall be
left for the next pass. A file it cannot stat or read shall be skipped for this pass.

### SPEC-0080-REQ-009: Exactly once per line

For each file, the exporter shall insert that chunk's rows and upsert
`ingest_file(path, pos = old pos + bytes consumed, updated = now())` **in the same
transaction**. If the transaction fails, neither the rows nor the offset move, and the same
bytes are read again next pass. Lines are never duplicated and never skipped.

### SPEC-0080-REQ-010: Unparsable lines

If a line is not JSON, or has no `t`, or `t` is not a number, the exporter shall drop that line
and still advance the offset past it.

### SPEC-0080-REQ-011: Marker de-duplication

When a `visor` line has `event == "marker_add"`, the exporter shall compute the key
`(type, name, round(rel[0]/5), round(rel[1]/5), round(rel[2]/5))` (5 m cells; `rel` defaults to
`[0, 0, 0]` when absent) and drop the line if that key has been stored before or already
appears earlier in the same chunk. The set of stored keys shall be rebuilt from
`SELECT data FROM event WHERE stream='visor' AND data->>'event'='marker_add'` the first time the
writer runs and after every error. Measured: the game re-adds the same markers every time an
area loads; on one day 40,468 of 41,760 `marker_add` lines were repeats.

### SPEC-0080-REQ-012: Trail rows carry their planet address

For each `trail` line the exporter shall set `event.address` to the line's `ua` packed as a
universe address (`docs/data-formats.md`, "Universe address"). If `ua` is null or unusable, it
shall instead use the address last seen for the same planet centre: the key is `planet_pos` with
each component rounded to the nearest integer, and it maps to the most recent packed address
whose planet field (bits 52–55) is non-zero. If neither is known, `address` is NULL. The centre
map shall be rebuilt from existing trail rows
(`SELECT DISTINCT ON (data->'planet_pos') data->'planet_pos', address ... ORDER BY data->'planet_pos', id DESC`,
planet bits non-zero) whenever the marker set is rebuilt. Why: the hook writes `ua: null` when
it was attached mid-game and could not find the galactic address; the planet centre is constant
for a planet within a session, so it identifies the planet.

### SPEC-0080-REQ-013: Other streams

For `visor`, `systems` and `weather` lines, `address` shall be NULL and `data` the line
unchanged.

### SPEC-0080-REQ-014: Errors keep the thread alive

If a pass raises, the exporter shall record `ingest.error = "<ExceptionType>: <message>"`, discard
the marker set and centre map, close this thread's connection, and continue the loop. On a
successful pass it shall clear `ingest.error` and set `ingest.last_ok` to now, and print
`ingest: N new rows` when N > 0.

### SPEC-0080-REQ-015: Incremental, paged reads

Readers shall fetch a stream with
`SELECT id, address, data FROM event WHERE stream = $1 AND id > $last AND t >= $since ORDER BY id LIMIT 50000`,
repeating until an empty page, and keep the last `id` as their cursor for the next call. The
trail reader's first call shall use `since = now - 2 days`; later calls `since = 0`.

### SPEC-0080-REQ-016: Labels and planet facts

`put_label(key, label)` shall upsert into `find_label` when `label` is non-empty and delete the
key otherwise. `put_planet(address, facts)` shall upsert `planet_facts` setting `updated = now()`.
`labels()` and `planets()` shall return the whole table as a dict.

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| `PGHOST` unset | No migration or ingest thread starts; the log says the history APIs will answer 503 (SPEC-0081). Day files keep growing on disk and are ingested in full once the store is configured. |
| Database unreachable at start | Migration retries every 30 s; `ingest.error` holds the reason; `/metrics` reports `nms_db_up 0`. |
| Database lost while running | The pass fails, the error is recorded, the connection is dropped, and the next pass (2 s later) reconnects. The writer lock is released with the dead session and retaken on reconnect. No line is lost: offsets did not move. |
| A second exporter running | It cannot take lock 4208, writes nothing, reads normally. |
| Game not running / no files | Passes find nothing to do; not an error. |
| A file truncated or replaced by the hook | Not handled: the offset would exceed the new size and the file is skipped. The hook only appends, and a new day starts a new file, so this does not occur in practice. |

## Verification

1. Start the exporter with a configured database and an empty live directory containing one
   `trail-YYYYMMDD.jsonl` of a few lines. Within 2 s:
   `SELECT path, pos FROM ingest_file;` shows the file with `pos` equal to its size, and
   `SELECT count(*) FROM event WHERE stream='trail';` equals its line count.
2. Append half a line (no newline). Count does not change; `pos` does not change. Finish the
   line; it appears once.
3. Restart the exporter. Counts do not change (exactly once across restarts).
4. Append the same `marker_add` line (same type, name, `rel` within the same 5 m cells) twice to
   a visor file: exactly one row appears in `SELECT * FROM marker`.
5. Write a trail line with `ua` set, then one with `ua: null` and the same `planet_pos`: both rows
   have the same `address` (`SELECT address FROM trail ORDER BY id DESC LIMIT 2`).
6. Run a second exporter on another port against the same database:
   `SELECT pid, application_name FROM pg_locks JOIN pg_stat_activity USING (pid) WHERE locktype='advisory' AND objid=4208;`
   shows exactly one holder, and row counts do not double.
7. Stop the database for a minute while appending lines, then start it: every line arrives once,
   and `nms_db_up` goes 1 -> 0 -> 1.

## Lessons

See `spec/lessons/SPEC-0099-storage-deploy-lessons.md`. Top two: commit the file offset in the
same transaction as the rows (anything else duplicates or loses lines on a crash), and
de-duplicate `marker_add` on the way in (97 % of them are the game re-adding markers it already
added).
