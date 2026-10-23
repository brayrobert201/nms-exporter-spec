---
id: SPEC-0030
title: 'Read game numbers from the installed build, not a wiki or a community dataset'
description: 'Every game constant the system uses is extracted from the user''s own installed paks, keyed by Steam build id, and re-extracted after updates.'
type: decision
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0020, SPEC-0021, SPEC-0022, SPEC-0026, SPEC-0012]
---

# SPEC-0030: Numbers from the installed build

## Context

The exporter projects farm output, refiner completion and power, and plans crafting, from
constants the save does not contain: recipe times, link-grid rates, hotspot strengths, mission
progress values. Three places offer those numbers:

1. **Wikis and forum posts.** Readily searchable, but often written before a major update
   (Origins, Frontiers, Worlds, Cosmos) and rarely say which version they describe. The main
   community wiki also refused automated fetches (HTTP 402) during the tooling survey, so it could
   not even be stored as a source.
2. **Community datasets** (e.g. the data behind recipe and price apps). Extracted from game files
   with the same tools, so a cross-check, not independent evidence; their update cadence is theirs.
3. **The installed game's own `.pak` files**, via HGPAKtool and MBINCompiler. Authoritative for
   the build the user is actually playing; requires two third-party binaries and a .NET runtime.

The numbers matter: a refiner ETA is wrong by a factor of 250 (substance outputs) or 2 (product outputs) if
`TimeToMake` is read as seconds, and the farm rate is wrong by 2.5x if the
hotspot class strength is guessed.

## Decision

All game constants shall be extracted from the user's installed paks:

- Tables go to `data/nms_tables.json` with `meta.game_build_id` and `meta.mbincompiler_version`
  (SPEC-0021); the runtime reads only that file.
- Ad-hoc lookups go through the build-keyed pak tool (SPEC-0020); any value relied on in the
  compendium is cited from the installed build with build, pak, path and line range (SPEC-0026).
- Where the data does not contain a number (network diminishing returns, per-sale price drop,
  frigate rank thresholds), the system shall either measure it or label it by provenance in the
  key name (`level_thresholds_community`), never present it as game data.
- After a game update, re-extract. The build id in the Steam manifest is the staleness key for
  tables, unpacked files, cited sources and compendium concepts.

## Alternatives considered

- **Ship a pre-extracted JSON from one build.** Simplest for a user, but numbers silently drift
  after the next update, and the file contains the game's text (item names, mission titles),
  which a public repo should not redistribute (SPEC-0032).
- **Scrape a wiki at runtime.** Version laundering, no build key, fetches refused, and the runtime
  gains a network dependency for numbers that sit on local disk.
- **Hardcode constants in code.** Fine for a handful; there are thousands (1,684 recipes, 5,235
  items, 499 missions).

## Consequences

- A user must install HGPAKtool, MBINCompiler (the release matching their game version) and a .NET
  8 runtime into a tools directory before the planner and farm rates work. The specs say so up
  front and every dependent panel degrades with an explanation when the tables are absent.
- MBINCompiler lags game updates by days. Until a matching release exists, the previous tables
  remain in use and the lint flags concepts as unchecked on the new build.
- Values that are configured in data but applied by code (Harvest All radius, storm ranges) still
  need measurement before they are trusted as behaviour; this decision makes the configured value
  reliable, not the behaviour (SPEC-0026-REQ-003).
- Revisit if Hello Games publishes an official data export, or if the pak format changes such that
  community tools stop keeping up.
