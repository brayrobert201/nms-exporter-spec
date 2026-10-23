---
id: SPEC-0002
title: 'The exporter service: snapshot, HTTP API and metrics'
description: 'A standard-library HTTP server that decodes the save on change, keeps a static snapshot, projects it per request, and serves /metrics, /json, /diff, /raw/bases and /healthz.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0001]
supersedes: []
superseded_by: []
related_specs: [SPEC-0003, SPEC-0004, SPEC-0005, SPEC-0006, SPEC-0007, SPEC-0008, SPEC-0009, SPEC-0010, SPEC-0011, SPEC-0012, SPEC-0021, SPEC-0060, SPEC-0081]
screenshots: []
---

# SPEC-0002: The exporter service: snapshot, HTTP API and metrics

## Summary

One long-running process on the gaming PC reads the save (SPEC-0001) and serves it to three kinds
of consumer: a Prometheus-compatible scraper (`/metrics`, feeding Grafana, SPEC-0008), the wall
screens (`/json`, SPEC-0009) and shell tools (`/diff`, `/raw/bases`). The work is split in two:
a **static snapshot** built once per new save (decode, match recipes, trace pipe networks), and a
**projection** applied on every request that moves time-dependent numbers forward to "now"
(SPEC-0003). A tiny state file keeps what must survive restarts: fill-level history for measured
extraction rates and a 48-save history for `/diff`. The map pages and their database-backed APIs
are served by the same process but belong to the `map-ui` and `storage-deploy` areas.

## Inputs and outputs

| Input | Default | Override |
|---|---|---|
| Listen address / port | `0.0.0.0:9877` | `NMS_EXPORTER_HOST`, `NMS_EXPORTER_PORT` |
| Game tables | `data/nms_tables.json` (built by the `game-data` area's extractor) | `NMS_TABLES` |
| Key mapping | see SPEC-0001-REQ-006 | `NMS_MAPPING` |
| State file | `$STATE_DIRECTORY/nms_state.json`, else `~/.cache/nms-exporter/nms_state.json` | `NMS_STATE` |
| Storage plan | `data/nms_storage_plan.json` (SPEC-0007) | `NMS_STORAGE_PLAN` |
| Planner targets | `ULTRAPROD1,ULTRAPROD2` (SPEC-0006) | `NMS_PLAN_TARGETS` |
| S3 backup | disabled unless keys set (SPEC-0010) | `NMS_S3_*` |

### Routes (this area)

| Route | Response |
|---|---|
| `GET /metrics` | Prometheus text format 0.0.4, every metric projected to now |
| `GET /json` | The projected snapshot as JSON (sets serialised as lists) |
| `GET /diff?against=N` | What changed between the latest save and the one N saves earlier (default 1) |
| `GET /raw/bases` | `PersistentPlayerBases` reduced to `Name, BaseType, GalacticAddress, Position, Forward, LastUpdateTimestamp, Objects` |
| `GET /healthz` | `ok`, text/plain, always 200 while the process serves |
| `GET /` | Plain-text route list for scripts; a request with `Accept: text/html` is redirected (302) to `/map` |
| anything else | 404 `not found` |

Every response sets `Content-Length` and `Cache-Control: no-store`; `HEAD` is served by the same
handler without a body. HTTP/1.1 keep-alive, `ThreadingHTTPServer` with daemon threads, access
logging off.

### `/json` top-level keys

`errors` (map, empty when healthy), `save`, `player`, `universe`, `stats`, `collections`, `items`,
`locations`, `refiners`, `farms`, `plants`, `planner`, `fleet`, `missions`, `tech`, `tidy`,
`blueprints`, `crafting`, `known`, `backup`, `now`. Field-level contents are in SPEC-0003 to
SPEC-0007 and SPEC-0010.

## Requirements

### SPEC-0002-REQ-001: Refresh on request, not on a timer

When a request arrives for `/metrics`, `/json`, `/diff`, `/raw/bases` or `/`, the server shall
call `refresh()` first. `refresh()` holds one lock, kicks the backup (SPEC-0010), finds the newest
save and re-decodes only if `(path, mtime)` changed (SPEC-0001-REQ-004). There is no background
poller; a scrape every 60 s is the heartbeat.

### SPEC-0002-REQ-002: Static snapshot built once per save

On a successful decode the exporter shall build the snapshot (all non-time-dependent fields) and
replace the previous one atomically (a single reference assignment). It shall then append to the
`/diff` history, write the state file, and start the crafting evaluation on a background thread
(SPEC-0006) so no scrape waits for it.

### SPEC-0002-REQ-003: Projection per request

Every `/json` and `/metrics` response shall be computed from a deep copy of the snapshot projected
to the request time (SPEC-0003), so concurrent requests never see a half-updated structure.

### SPEC-0002-REQ-004: Errors map

The snapshot shall carry `errors: {source: message}`. Each independent part (save, planner,
crafting, history, state, tidy) catches its own exception and records it there; one failing part
shall not prevent the others from rendering. Healthy means `errors == {}`.

### SPEC-0002-REQ-005: State file

The exporter shall keep `nms_state.json` as `{"farms": {farm_key: [[anchor_ts, stored_units],
...]}, "history": [...]}`. It is written via a temp file and `os.replace` after each new save.
If it cannot be read at start, the exporter starts with `{"farms": {}}`; if it cannot be written,
`errors.state` says why.

### SPEC-0002-REQ-006: Save history for /diff

For each decoded save the exporter shall record
`{timestamp, file, units, nanites, items: {id: total}, farms: {key: round(stored_items)}}`,
replacing the last record when `(timestamp, file)` is unchanged, and keep the newest **48**.

### SPEC-0002-REQ-007: /diff

`/diff?against=N` shall clamp N to `[1, recorded-1]` and return
`{from, to, units{from,to,delta}, nanites{...}, items[{id,item,from,to,delta}], farms[{farm,from,to,delta}], recorded, against}`
with items sorted by `|delta|` descending and only changed rows included. With fewer than two
records it shall return `{"error": "fewer than two saves recorded yet", "recorded": n}`. A
non-integer `against` is treated as 1.

### SPEC-0002-REQ-008: Metric families

The exporter shall emit, with `# HELP` and `# TYPE` lines, at least these families (labels in
braces). Rows whose value is `None` are omitted.

| Group | Metrics |
|---|---|
| Collector | `nms_collector_up`, `_decodes_total`, `_decode_failures_total`, `_decode_duration_seconds` |
| Backup | `nms_backup_enabled{bucket}`, `_uploads_total`, `_errors_total`, `_last_success_seconds` |
| Save | `nms_save_info{file,summary,version,platform,mode,difficulty,context}`, `nms_save_mtime_seconds`, `nms_save_timestamp_seconds`, `nms_save_age_seconds` |
| Player | `nms_units`, `nms_nanites`, `nms_quicksilver`, `nms_health`, `nms_shield`, `nms_life_support`, `nms_ship_health`, `nms_time_alive_seconds`, `nms_play_time_seconds`, `nms_ammo{weapon}` |
| Where | `nms_universe_address{axis}`, `nms_universe_info{glyphs,address,galaxy,system,planet}` |
| Counts | `nms_collection_count{kind}`, `nms_stat{stat}` (~470 lifetime counters) |
| Inventory | `nms_item_total{item,id,type}`, `nms_item_stacks{...}`, `nms_item_amount{location,id,item,type}` |
| Refiners | `nms_refiner_*{refiner,base,recipe,output}`: active, progress_ratio, remaining_seconds, finish_timestamp_seconds, output_amount, output_saved, input_amount{+input}, seconds_per_op, last_update_seconds |
| Farms | `nms_farm_*{base,name,kind}`: stored_items, stored_items_saved, capacity_items, fill_ratio, rate_items_per_hour, rate_measured, full_in_seconds, extractors, depots, anchor_seconds |
| Plants | `nms_plants_total{base}`, `nms_plants_mature{base}`, `nms_plant_ready_in_seconds{base,plant,index}`, `nms_plant_kind_{total,mature,next_ready_seconds,stock}{base,plant,yields}` |
| Fleet | `nms_frigates{class}`, `nms_expedition_*{expedition,category,duration}`, `nms_frigate_{score,level,stat,expeditions,info}` |
| Missions | `nms_missions{status}`, `nms_mission_remaining_steps`, `nms_mission_info{mission,title,objective,rewards,selected}`, `nms_mission_stages` |
| Technology | `nms_tech_{installed,supercharged,supercharged_wasted,supercharged_good,isolated,broken,info,move,isolated_family}` |
| Tidy | `nms_tidy_{moves,merges,stacks_after,capacity,stacks_now,incoming,outgoing,move}` (only when the plan loaded) |
| Crafting | `nms_crafting_compute_seconds`, `nms_craftable`, `nms_craftable_direct`, `nms_recipe_yield`, `nms_recipe_need`, `nms_craft_raw_need`, `nms_craft_shortfall`, `nms_craft_blocked_{by,only_by,have,short}` |

Crafting metrics carry both `id`/`recipe_id` (stable join keys) and `item`/`recipe` (display).

### SPEC-0002-REQ-009: Label escaping

Label values shall have every non-printable character replaced by `?`, then `\` and `"` escaped.
Procedural item ids contain bytes that a text parser reads as line breaks; unescaped, one such id
invalidates the whole scrape.

### SPEC-0002-REQ-010: Down means down, but still answer

If there is no valid save (`errors.save` set or no `save` section), `/metrics` shall emit
`nms_collector_up 0` plus the collector and backup families and nothing else, so a dashboard can
tell "exporter down" from "exporter up, save unreadable".

### SPEC-0002-REQ-011: Request errors

An exception while serving shall produce `500 error: <message>` (text/plain); a client that hung
up (`BrokenPipeError`) is ignored. The process shall not exit on a request error.

## Degraded behaviour

- **Tables missing or unreadable**: the snapshot still builds; names fall back to raw ids, recipes
  and farms caps fall back to built-in defaults (1440 units/item, 360,000 / 1,440,000 storage,
  100 units/s), missions report `tables: false` and every mission `unknown`.
- **State directory read-only**: rates fall back to assumed (SPEC-0005), `/diff` resets on
  restart, `errors.state` says why.
- **Crafting still computing**: `crafting.computed_at` is `null` and the crafting metric families
  are empty for a few seconds after each new save; everything else is current.
- **Save unreadable**: SPEC-0001 degraded behaviour; the last good snapshot stays.
- **Database for the map down**: not this area; `/metrics` and `/json` are unaffected and gain
  `nms_db_up 0`.

## Verification

```bash
curl -fs localhost:9877/healthz                              # ok
curl -s localhost:9877/json | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d["errors"], d["save"]["file"], round(d["save"]["age"]))'
curl -s localhost:9877/metrics | grep -c '^nms_'             # thousands of lines
curl -s localhost:9877/metrics | promtool check metrics      # no parse errors (warnings about naming are fine)
curl -s 'localhost:9877/diff?against=3' | python3 -m json.tool | head -40
```

Run a second instance against the working tree on a scratch port (`NMS_EXPORTER_PORT=9878
NMS_STATE=/tmp/x/state.json`) before installing; the deploy script fails if `/json` has a
non-empty `errors` map (SPEC-0011).

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): label values with raw bytes broke
the scrape; quoted item names broke Grafana label matchers, so every metric carries an id label.
