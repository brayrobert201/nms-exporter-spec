# Reproducing the system

The build order across all five areas, written for a coding agent working with a person who owns
the game. Each step names the specs to read, what to build, and what must be true before moving
on. Do the steps in order: each one's verification is the next one's input. Shared formats are in
`docs/data-formats.md`; hook line types in `docs/hook-events.md`; game-file locations in
`docs/game-data-locations.md`.

| Step | Area | SPEC range |
|---|---|---|
| 1 | save-exporter | 0001–0019 |
| 2 | game-data | 0020–0039 |
| 3 | live-hook | 0040–0059 |
| 4 | map-ui | 0060–0079 |
| 5 | storage-deploy | 0080–0099 |

Steps 1 and 2 are useful alone (metrics, wall screens, Grafana). Steps 3–5 add the live map.
Step 5 can be done in its minimal form (one Postgres anywhere on the LAN, env file by hand) or
fully (managed cluster database, machine identity, LAN name).

## Ground rules for the agent

- **Read the specs first, then the decisions.** Decisions record what was tried and rejected;
  several of them exist precisely so that a later builder does not "fix" a deliberate choice
  (no file fallback for the map, SPEC-0090; standard library on the save side, SPEC-0012).
- **Nothing from the game goes in the repository.** Icons, glyphs, tables and footprints are
  extracted on the user's machine from their installed copy (SPEC-0032).
- **No secrets in the repository.** Every credential arrives through an env file written at
  deploy time (SPEC-0082, SPEC-0084).
- **Every fetch returns a value or an error and never takes the process down.** One dead source
  must not blank a screen; errors surface in the payload's `errors` map.
- **Look at what you render.** Layout is what breaks. Screenshots of the map and the wall screens
  at their real sizes are part of every verification.
- **Ask the person before anything that touches the running game** (attaching the hook, live
  patches). Ask them to save first. A crashed game costs their time, not yours.

## Prerequisites

| Need | For | Notes |
|---|---|---|
| Linux desktop with Steam and No Man's Sky under Proton | everything | the exporter must run on this machine, as the Steam user |
| Python ≥ 3.12 at `/usr/bin/python3` | everything | the save side uses only the standard library |
| systemd | services and timers | |
| `uv` | step 5 | OS package, or the standalone binary for a quick test |
| hgpaktool, MBINCompiler, a .NET 8 runtime | step 2 | installed into a tools directory, not the repo (SPEC-0020) |
| a Prometheus-compatible scraper and Grafana | step 1 dashboards | optional |
| conky (`conky-all`), Xvfb for test renders | step 1 wall screens | optional |
| an S3-compatible bucket | step 1 save backup | optional (SPEC-0010) |
| a Windows Python inside the game's Proton prefix, pyMHF, NMS.py | step 3 | SPEC-0040 |
| PostgreSQL 17 reachable on the LAN, not on the gaming PC | step 5 | minimal form: any always-on machine |
| a Kubernetes cluster with a Postgres operator (e.g. CloudNativePG), an external-secrets operator, a LAN load balancer (e.g. MetalLB), a reverse proxy (e.g. Traefik) | step 5 full form | optional |
| a secret store with AppRole-style machine roles (OpenBao or Vault) | step 5 full form | optional |
| a LAN DNS resolver you can add host overrides to | step 5 LAN name | optional |

## Step 1: save exporter (SPEC 0001–0019)

Read SPEC-0001 (save decoding), SPEC-0002 (exporter service), SPEC-0003 (projection), then
SPEC-0004 to SPEC-0007 for each section of the snapshot, SPEC-0011 (install and service), and the
decisions SPEC-0012 to SPEC-0015.

Build:

1. The save decoder: find the newest `save*.hg` under the Proton prefix, decode its LZ4 chunk
   chain, de-obfuscate keys with the MBINCompiler mapping.
2. The HTTP exporter on `:9877`: `/metrics`, `/json`, `/diff`, `/raw/bases`, `/healthz`.
3. Projection, refiners, farms, plants, fleet, inventory, missions, technology.
4. The systemd template unit `nms-exporter@<user>` and `install.sh` (SPEC-0011). At this stage the
   unit can run `/usr/bin/python3`; step 5 switches it to the venv.
5. Optional: Grafana dashboards (SPEC-0008), conky wall screens (SPEC-0009), save backup to S3
   (SPEC-0010).

Verify (details in SPEC-0001 and SPEC-0002):

- `curl -fs localhost:9877/healthz` prints `ok`.
- `/json` has an empty `errors` map, and `save.file` names the newest save.
- `nms_units` matches the in-game balance; `nms_collector_up 1`.
- Save in game: within one scrape `nms_save_mtime_seconds` advances.
- If built: the wall screens rendered in Xvfb at `-dpi 96` and looked at.

Do not continue until numbers on `/json` match what the game shows.

## Step 2: game data (SPEC 0020–0039)

Read SPEC-0020 (pak access), SPEC-0021 (tables), SPEC-0022 (reference values), SPEC-0023 (icons
and glyphs), SPEC-0024 (part footprints), and decisions SPEC-0030 to SPEC-0032. SPEC-0025
(a loose-file mod) and SPEC-0026 (the compendium method) are optional.

Build:

1. The pak index and unpack/convert tool.
2. The table extractor writing `data/nms_tables.json` (recipes, items, hotspot and extractor
   constants, missions) keyed by the Steam build id.
3. The icon and glyph extractors (`data/icons/`, `data/glyphs/`, git-ignored).
4. The footprint extractor for base parts (used by the map's base view).

Verify:

- The extractor's summary line and the spot values in SPEC-0021's table match on your build (or
  differ only where the game changed; re-check against the game).
- `/icon/<name>.png` and `/glyph/0.png` are served by the exporter; `/glyph/0.png` is the glyph
  the game calls Sunset.
- Step 1's refiner and farm numbers still match the game with the extracted tables in place.

## Step 3: live hook (SPEC 0040–0059)

Read SPEC-0040 (pyMHF under Proton: launch mode and attach mode), SPEC-0041 (executor hot-patching), SPEC-0042 (live
position), SPEC-0043 (frames, heading, radius, address), SPEC-0044 (weather), SPEC-0045 (visor
log and chat commands), SPEC-0046 (galaxy log), decisions SPEC-0047 and SPEC-0048, and the lessons
SPEC-0049 **before** attaching anything. Formats: `docs/hook-events.md`.

Build:

1. The Windows Python in the prefix; the Steam launch-options wrapper (launch mode, preferred:
   pyMHF starts the game paused and injects before boot); and the attach launcher for a game
   already running, once per run.
2. The three mods: position (`live.json` + trail), visor log, galaxy log.
3. In the exporter: reading `live.json` (state; returns a value or an error, never raises).

Verify (with the person at the game, after they save):

- `live.json` updates every 0.25 s; walking 10 m adds about five trail lines.
- Walking north on the in-game compass raises `rel.y`; facing east gives a heading near 90°.
- Standing in a base, `|rel|` is within tens of metres of that base's saved `Position` magnitude.
- A visor survey produces `marker_add` lines; `/hs S silver` in game chat produces a `label`
  line.
- Detaching leaves the game running (attach mode). In launch mode, the pyMHF log shows
  "injection complete" *before* the first `New State:` line, and the first `live.json` already
  carries `ua`.

At this stage the day files exist and grow; nothing reads their history yet. That is expected:
the map's history comes from step 5's store.

## Step 4: map UI (SPEC 0060–0079)

Read SPEC-0060 (page, frames, `/api/map`), SPEC-0061 (live layer), SPEC-0062 (heights),
SPEC-0063 (finds), SPEC-0064 (interaction), SPEC-0065 (globe), SPEC-0066 (base footprints and
power), SPEC-0067 (sectors and places), SPEC-0068 (systems page), SPEC-0069 (design rules), and
decisions SPEC-0070 to SPEC-0072.

Build in two passes:

1. **Save-only first:** `/map` with planets, bases and base layouts from `/api/map`. This needs
   only steps 1–2 and works without any database.
2. **History second**, after step 5's store exists: live layer, trail, heights, finds, sectors,
   systems page. These read through the store's `tail()` (SPEC-0080 REQ-015). Build the
   "Database down" banner (SPEC-0081) at the same time, not later.

Verify:

- The planet view centres between two bases and their distances from centre are `R x angle`.
- In game, face north and walk: the arrow points up and the trail runs up the screen.
- At about 412 x 900 (a phone) nothing scrolls sideways; screenshot it.
- With the database stopped, `/map` still shows the save's planets and bases and a red
  "⚠ Database down" pill with the reason.

## Step 5: storage and deploy (SPEC 0080–0099)

Read SPEC-0080 (store and ingest), SPEC-0081 (Postgres only), SPEC-0082 (environment and
service), SPEC-0083 (managed Postgres), SPEC-0084 (machine identity), SPEC-0085 (LAN name),
decisions SPEC-0090 to SPEC-0092, and lessons SPEC-0099.

Minimal form (one Postgres, env file by hand):

1. Create database `nms` owned by role `nms` on an always-on LAN machine that is not the gaming
   PC; require TLS.
2. Add `pyproject.toml` and `uv.lock` (SPEC-0082); change the unit to run `.venv/bin/python`;
   make `install.sh` run `uv sync --frozen --no-dev --python /usr/bin/python3` with
   `UV_PYTHON_DOWNLOADS=never`.
3. Write `/etc/nms-exporter/db.env` (root, 600) with the six `PG*` variables by hand.
4. Implement `db.py`: schema under lock 4207, ingest every 2 s under lock 4208, offsets in the
   same transaction, marker de-duplication, trail addresses, 50,000-row paging (SPEC-0080).
5. Implement the 503 handling and `nms_db_up` (SPEC-0081), with the "configured" check before any
   driver call.

Full form adds, in this order:

6. Create the database secret in the secret store (`db_username`, `db_password`) **first**.
7. Push the operator-managed cluster: 2 instances off the gaming PC, LB service for the rw
   primary, daily snapshot, WAL archive, external secret for bootstrap (SPEC-0083).
8. Create the host's machine role and policy, mint and unwrap its secret ID, install the sync
   timer (SPEC-0084). Bind the role to the address the audit log actually shows.
9. The LAN name: selectorless Service, hand-applied EndpointSlice, Ingress, DNS override
   (SPEC-0085).

Verify:

- `curl -s localhost:9877/metrics | grep nms_db_up` prints `nms_db_up 1`.
- `SELECT path, pos FROM ingest_file` lists each day file with `pos` equal to its size within
  a few seconds of a write; restarting the exporter does not change row counts.
- Repeated `marker_add` lines produce one row in the `marker` view.
- A second exporter on port 9878 against the same database does not double rows (one holder of
  advisory lock 4208).
- With no `PGHOST`, history routes return 503 `{"error": "nms-db not configured: ...", "db": false}`
  and the log never mentions a Unix socket.
- Stop the database: `nms_db_up 0`, banner on `/map`, `/metrics` and `/json` unaffected. Start it:
  nothing missing.
- Full form: `pg_stat_archiver` shows `archived_count` rising and `failed_count` not rising; 20
  logins from the gaming PC all come from one address in the audit log; a login with the
  copied credentials from another machine fails with the CIDR error; `https://<map name>/`
  redirects to `/map` from a LAN client and does not resolve from outside.

## After the build

- **Every deploy:** run the working tree on 9878 first (the dev-deploy script), check `errors` is
  empty, then install. The test instance shares the database read-only.
- **After a game update:** re-extract the tables, icons and footprints (step 2); expect the hook's
  signatures to need checking before attaching (step 3).
- **After an OS upgrade:** rerun `install.sh` so the venv is rebuilt against the new Python.
- **Rotating the database password:** `ALTER ROLE` first, then the secret store; the sync timer
  carries it to the gaming PC within 6 hours (or start the sync service by hand).
