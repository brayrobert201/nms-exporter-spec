---
id: SPEC-0048
title: 'The mods write raw JSONL day files; interpretation lives in the exporter'
description: 'Hook output is an append-only raw record written by the mods inside the game (live.json plus trail/weather/visor/systems day files); the exporter reads, ingests and interprets it and never asks the game for anything.'
type: decision
area: live-hook
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0041, SPEC-0042, SPEC-0043, SPEC-0044, SPEC-0045, SPEC-0046]
---

# SPEC-0048: The mods write raw JSONL day files; interpretation lives in the exporter

## Context

Code inside the game is expensive to change (one attach per run, SPEC-0047), dangerous when wrong
(a bad read can crash the game, an exception disables a detour), and runs on the game's frame
thread. Code in the exporter is cheap to change and restart. Several interpretations were wrong at
first and corrected later from data already recorded: the ship's heading sign, the planet's nominal
radius, storm timing for Extreme planets, which planet a marker belongs to, whether a hotspot's class
is in the marker. Each correction needed the raw values, not a derived number.

## Decision

- **The mods write files; nothing reads the game.** Each mod writes, under
  `C:\pymhf\mod_saves\live\` in the prefix:
  - `live.json` - the latest position sample, replaced atomically (temp file + `os.replace`);
  - append-only day files `trail-`, `weather-`, `visor-`, `systems-YYYYMMDD.jsonl`, one JSON object
    per line, every line stamped with `t` (Unix seconds) and (except trail/weather) an `event` name.
  Formats in `docs/hook-events.md`.
- **Record raw, interpret later.** Mods write the game's own values (world vectors, `at` as given,
  enum names or raw ints, seeds, unpacked addresses, even whole structs as hex for hotspot markers)
  with only cheap, reversible derivations (`rel = pos - planet_pos`). Sign conventions, radii, storm
  phases, portal codes, nearest-hotspot matching and planet assignment all live in the exporter.
- **Mods rate-limit and dedupe at source** (4 Hz position, trail only on > 2 m moves, marker key
  repeat 60 s, systems once per session, names once per (seed, name)), because the game calls some
  hooks thousands of times a frame and the files must stay small enough to keep.
- **The exporter is a reader.** It polls `live.json` (stale after 10 s) and ingests new day-file lines
  into its own store (storage-deploy area) for history; it never connects to the executor.

## Alternatives considered

- **Mod serves an HTTP API (pyMHF `http_api`).** Live but no history; the exporter would need the game
  up to show anything, and an exporter bug could make requests into the game.
- **Mod writes straight to the database.** Adds a network client and credentials inside the game
  process; a slow database would stall a frame hook.
- **Mod computes map-ready values (heading, lat/lon, storm ETA).** Every correction would need a hot
  patch or an attach; old records would carry the wrong numbers forever.
- **One growing file per stream.** Day files bound size and make a day easy to drop or re-ingest.

## Consequences

- Corrections apply retroactively: re-reading the trail with the ship sign or the modal radius fixed
  every past sample.
- The record is useful without the exporter (grep, jq) and survives exporter and database outages:
  the exporter catches up by reading the files.
- Writes happen on the game thread; they are small appends and one ~1.5 kB atomic replace every
  0.25 s. A full disk raises inside the detour and pyMHF disables it.
- Files grow: the first evening's `visor-*.jsonl` reached ~17.8 MB (40k `marker_add` lines, mostly
  deposits) after dedup; mute more types if that matters.
- Positions in visor rows use the player's last 1 s context for the planet centre; marker rows have
  no address and depend on the trail timeline for planet assignment.
