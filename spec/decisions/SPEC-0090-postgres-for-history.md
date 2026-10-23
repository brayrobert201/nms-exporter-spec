---
id: SPEC-0090
title: 'Postgres for the map''s history, with no file fallback'
description: 'Why the live hook''s history moved from JSONL files read at startup to a managed Postgres, why not SQLite, and why the map fails loudly rather than falling back to files.'
type: decision
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0080, SPEC-0081, SPEC-0083, SPEC-0091, SPEC-0048, SPEC-0012]
---

# SPEC-0090: Postgres for the map's history, with no file fallback

## Context

The live hook (SPEC 0040–0059) writes four kinds of JSONL day file inside the game's Proton
prefix: trail (a sample whenever the player moves more than 2 m), visor (markers and visor
events), systems (every star system the galaxy map classifies) and weather (storm
transitions). Before this decision the exporter built the map's history by reading **every day
file in full on every start**, then tailing them. Two small JSON files beside them held what the
player recorded on the map (hotspot classes) and per-planet facts.

Problems, measured on the original:

- **Start-up cost grows forever.** Every restart (and every deploy restarts) re-parsed all
  history. One day's visor log was 17.8 MB, mostly the game re-adding markers it had already
  added: 40,468 of 41,760 `marker_add` lines that day were repeats.
- **Not backed up.** The files live inside the Proton prefix in the user's Steam directory. The
  save backup (SPEC 0001–0019) covers saves, not this. Deleting or rebuilding the prefix (a
  common fix for Proton problems) would take all history with it.
- **No ad-hoc queries.** "Where was I at 21:00 yesterday" meant a script.

The history was still small (MBs per day), and the hook was still changing shape. The owner's
framing was that this was the cheapest moment to move: "move early rather than late", before
there was a year of files to migrate and more readers depending on the file layout.

## Decision

1. **Postgres is the store** for everything that is not in the save: one `event` table with
   `stream`, `t`, `address` and the hook's line as `jsonb`, plus `ingest_file` offsets,
   `find_label` and `planet_facts` (SPEC-0080). The exporter copies new complete lines every
   2 s, exactly once, with the offset committed in the same transaction as the rows.
2. **The hook's files stay** as the raw output and the store's input. The hook does not talk to
   the database: it runs inside the game under Wine with its own Windows Python, and a network
   dependency in a game-frame hook is the wrong place for one.
3. **Postgres only, no fallback.** Readers never read the day files. When the database is
   unconfigured or down, history endpoints answer 503 with `{"error": ..., "db": false}`, the
   map shows a "Database down" banner, and `/metrics` carries `nms_db_up 0` (SPEC-0081). Save
   metrics and wall screens are unaffected.
4. **A managed, in-cluster Postgres**, two instances, off the gaming PC, backed up daily plus
   WAL archive (SPEC-0083).

The owner's reasoning on point 3 and 4: a database inside the home network, run by the same
cluster as everything else, is not an "external dependency" in the sense the save side's
standard-library-only rule guards against; it is infrastructure that is already there, backed
up and monitored. A fallback path would be a second implementation of every reader, used only
during outages, and therefore broken when needed; an honest error is better than stale or
partial history presented as current. This is written down so nobody later "fixes" the 503 into
a silent fallback.

## Alternatives considered

- **Keep the files, add an index or a cache file.** Fixes start-up time, not backup or
  queries; adds a third format to keep consistent.
- **SQLite on the gaming PC.** No new dependency (it is in the standard library), and it would
  remove the start-up cost and give SQL. Rejected: it would still live on the one machine that
  reboots at will and is not backed up for this data; a backup would have to be built for it;
  and a second reader (the test instance, ad-hoc queries from another machine) means file
  locking across processes on a desktop. The cluster already runs Postgres with backups and WAL
  archiving, so the marginal cost of one more database was small.
- **Postgres with a file fallback.** Rejected as above: two code paths for every reader, one of
  them only exercised during outages.
- **The hook writes to Postgres directly.** Rejected: it would need a Postgres driver inside the
  game's Windows Python, credentials inside the Proton prefix, and network calls on the game's
  frame thread. Files are the safest interface out of a hooked game.
- **A Postgres instance on the gaming PC.** Rejected: it is the only client and reboots at will,
  so the database would be down exactly when the exporter restarts.

## Consequences

- The exporter gains one third-party dependency (psycopg), which forced a Python environment
  (SPEC-0091).
- The gaming PC needs a database login, which forced machine identity for secrets (SPEC-0084,
  SPEC-0092).
- During a database outage the map shows the save's planets and bases and a banner, nothing
  more; the live position also disappears because `/api/live` returns trail and finds in the
  same response. Accepted because the database runs two instances.
- Start-up is constant time: readers page from the database by id (50,000 rows per query) and the
  trail reader only loads the last two days.
- History is backed up off the gaming PC with point-in-time recovery.
- Ad-hoc SQL works through the typed views (`trail`, `marker`, `chat`, `star_system`, `storm`).
- Repeated markers are dropped on ingest, so the stored visor stream is a few percent of the
  file's size.

Revisit if: the cluster is retired (move the same schema to a single Postgres on another always-on
machine; the exporter only needs libpq variables); the hook's output grows past what a 2 s
single-writer loop can keep up with (batch by `COPY`); or a user without any always-on machine
wants the map (then SQLite on the gaming PC, with the same schema, is the fallback design, chosen
deliberately rather than as an outage path).
