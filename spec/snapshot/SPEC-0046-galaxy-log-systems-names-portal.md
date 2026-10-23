---
id: SPEC-0046
title: 'galaxy_log mod: star systems, system and planet names, portal codes'
description: 'pyMHF mod that logs each star system the galaxy map classifies (economy, wealth, conflict, race, star, planets, PlanetSeeds), names with their seeds, galaxy and system map events; and how system records, names, portal glyph codes and KnownPortalRunes fit together.'
type: snapshot
area: live-hook
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0040, SPEC-0043]
supersedes: []
superseded_by: []
related_specs: [SPEC-0041, SPEC-0048, SPEC-0049]
screenshots: []
---

# SPEC-0046: galaxy_log mod: star systems, system and planet names, portal codes

> Runs inside the game process (SPEC-0040). Save before attaching.

## Summary

The save knows where the player's bases are but not what the surrounding systems are like or what
anything is called. Whenever the galaxy map draws a system the game classifies it
(`cGcGalaxyAttributesAtAddress::Classify`), filling an inline record of economy, wealth, conflict,
race, star type, planet count and planet seeds. Names are generated from seeds by
`cGcNameGenerator`. `galaxy_log.py` logs each system once per session and each (seed, name) once, so
the exporter can name planets and systems on the map and list every system seen with a dialable
portal code. Output: `systems-YYYYMMDD.jsonl` (fields: `docs/hook-events.md`).

## Inputs and outputs

| Hook | Event |
|---|---|
| `cGcGalaxyAttributesAtAddress.Classify.after(this, lUA)` | `system` (once per UA per session) |
| `cGcGalaxyMap.Data.DoSolarPopup.after(this, lQueryResult)` | `popup` when the hovered system changes |
| `cGcNameGenerator.GenerateSolarSystemName.after(this, lu64Seed, lResult, lLocResult)` | `system_name` |
| `cGcNameGenerator.GeneratePlanetName.after(this, lu64Seed, lResult, lLocResult)` | `planet_name` |
| `cGcFrontendPageSolarSystemMap.Update.after(this, lfTimeStep)` | `system_map_open` |
| `cGcPlayerEnvironment.Update.after` | `system_map_closed` (no sysmap Update for `MAP_IDLE_S = 0.75` s) |
| `cGcFrontendPageSolarSystemMap.OnMarkerPlaced.after(this, lpMarker)` | `system_map_marker` |

## Requirements

### SPEC-0046-REQ-001: Classify fast path
`Classify` can run for thousands of systems per frame while the galaxy map is open. The detour shall
first do `ua = int(lUA)` and return at once if `ua` is in the session's seen-set or `this` is null;
otherwise add it and process. Nothing else may run before that check.

### SPEC-0046-REQ-002: System record
If `this.contents.mbValid`, read `s = mStar` (`cGcGalaxyStarAttributesData`, inline) and write
`system` with: `ua`, `ua_hex` (`0x%X`), the unpacked address (`voxel` [x, y, z] signed, `system`,
`planet`, `reality`; SPEC-0043 REQ-008), and

| Field | Source | Enum |
|---|---|---|
| `economy` | `TradingData.TradingClass` | `cGcTradingClass` |
| `wealth` | `TradingData.WealthClass` | `cGcWealthClass` |
| `conflict` | `ConflictData` | `cGcPlayerConflictData` |
| `race` | `Race` | `cGcAlienRace` |
| `star` | `Type` | `cGcGalaxyStarTypes` |
| `anomaly` | `Anomaly` | `cGcGalaxyStarAnomaly` |
| `planets`, `prime_planets`, `space_pois` | `NumberOfPlanets`, `NumberOfPrimePlanets`, `NumberOfSpacePois` | int |
| `planet_seeds` | `PlanetSeeds[i].Seed` for i < min(planets, 16), masked to unsigned 64-bit | |
| `abandoned`, `pirate`, `gas_giant`, `giant`, `safe` | `AbandonedSystem`, `IsPirateSystem`, `IsGasGiantSystem`, `IsGiantSystem`, `IsSystemSafe` | bool |

Enums decode with the safe decoder of SPEC-0045 REQ-005 (name or raw int). Classify's `lUA` has
planet bits 0: it is the system's address.

### SPEC-0046-REQ-003: Names
`system_name` and `planet_name` rows shall carry `seed` (unsigned 64), `name` (`lResult.contents`
as text, stripped; null if empty) and `loc_name` (`lLocResult`, usually null). A `system_name` also
carries `popup_ua`, the UA of the last `popup` if it was under 1.0 s ago. Each `(event, seed, name)`
is written once per session (`GeneratePlanetName` runs every frame for the current planet).

### SPEC-0046-REQ-004: Seeds join names to systems and planets
The exporter shall use: a **system name's seed is the system's universe address** (planet bits 0);
a **planet name's seed is one of its system's `PlanetSeeds`**, and the planet with address planet
bits `n` (1-based) is named from `PlanetSeeds[n - 1]` (seen once: address planet 1 named from seed
index 0 while on it). Planet bits 0 have no name (SPEC-0043 REQ-007).

### SPEC-0046-REQ-005: Never raise
Every hook body except Classify's fast path shall run inside a guard that, on the first exception per
hook name, logs it and writes `{"event": "error", "where": <hook>, "error": repr(e)}`, then swallows
later ones.

### SPEC-0046-REQ-006: Portal code
The exporter shall render a 12-glyph portal address as hex digits **P SSS YY ZZZ XXX** from a
universe address: planet (4 bits), system index (12), Y (8), Z (12), X (12), each wrapped
two's-complement (the address without its reality byte):

```python
f"{p or 1:X}{(ua >> 40) & 0xFFF:03X}{(ua >> 24) & 0xFF:02X}{(ua >> 12) & 0xFFF:03X}{ua & 0xFFF:03X}"
```

Planet 0 becomes 1 (a portal lands on a planet). For a system with one of the player's planetary
bases, use that base's planet. Hex digit n is glyph n, in this order: 0 Sunset, 1 Bird, 2 Face,
3 Diplo, 4 Eclipse, 5 Balloon, 6 Boat, 7 Bug, 8 Dragonfly, 9 Galaxy, A Voxel, B Fish, C Tent,
D Rocket, E Tree, F Atlas. (Glyph images come from the game files; game-data area.)

### SPEC-0046-REQ-007: Known glyphs
The save's `KnownPortalRunes` is a bitmask, bit n = glyph n learned (value 15 = Sunset, Bird, Face,
Diplo; confirmed in game). For each system, if the base's planet glyph is unknown, the exporter shall
pick the first planet `1..min(planets, 15)` whose glyph is known; then report `dialable` = every
glyph of the code known, and `missing_glyphs`.

### SPEC-0046-REQ-008: Galactic coordinates and distance
Signal-booster style coordinates shall be `XXXX:YYYY:ZZZZ:SSSS` = `x+0x7FF : y+0x7F : z+0x7FF :
system`, 4 hex digits each. Distance between systems = voxel distance x 400 ly (a community rule, not
from game data; systems in one voxel show as the same region).

### SPEC-0046-REQ-009: Systems listing
The exporter shall list every system seen (latest `system` row per system address) with: name,
economy, wealth, conflict, race, star, anomaly, planets, the planet names in seed order, pirate,
abandoned, whether the player is in it now, distance in ly from the player's system, voxel, the
player's base names there (planetary bases only), portal code, galactic coordinates, `dialable` and
`missing_glyphs`; sorted by distance (unknown last), then name.

## Worked example

A made-up planet: voxel x = -5, y = 3, z = 100, system 0x2A, planet 2, reality 0.

| Step | Value |
|---|---|
| Packed universe address (SPEC-0043 REQ-008) | `0x202A0003064FFB` = 9053378793852923 |
| System address (planet bits cleared) | `0x2A0003064FFB`; this is also the system name's seed |
| Portal code P SSS YY ZZZ XXX | `2 02A 03 064 FFB` -> `202A03064FFB` |
| Glyphs, digit by digit | Face, Sunset, Face, Voxel, Sunset, Diplo, Sunset, Boat, Eclipse, Atlas, Atlas, Fish |
| Galactic coordinates | `07FA:0082:0863:002A` |
| Planet name seed | `planet_seeds[1]` of the `system` row for `0x2A0003064FFB` |
| Dialable with `KnownPortalRunes = 15` (glyphs 0-3 known) | no; `missing_glyphs` = 4, 6, A, B, F |

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| Galaxy map never opened this session | Only the current system (and neighbours the game classifies) appear; names only for what was generated. |
| Name not seen | Map falls back to "Planet n in system s". |
| Seed index unknown (planet bits > len(seeds)) | No planet name. |
| A hook raised | One `error` row; that hook's remaining events lost until re-enabled (SPEC-0041). |
| Mod loaded mid-session (live load) | Systems classified before load are not logged until seen again. |

## Verification

1. Open the galaxy map and hover a system: one `popup` with its `ua`, a `system_name` with
   `popup_ua` equal to it, and `seed == ua` for that system.
2. Arrive at a planet: a `planet_name` whose `seed` appears in the system row's `planet_seeds` at
   index `planet bits - 1`.
3. Compare a base's portal code with the in-game portal/glyph display for that planet.
4. A save with `KnownPortalRunes = 15`: systems whose code uses only digits 0-3 are dialable.

## Lessons

See [SPEC-0049](../lessons/SPEC-0049-live-hook-lessons.md): planet 0 treated as a planet; per-frame
name generation flooding the log until deduped by (seed, name).
