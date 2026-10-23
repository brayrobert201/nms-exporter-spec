---
id: SPEC-0044
title: 'Weather, storms and the current planet''s generation data'
description: 'live_position''s cGcSky and cGcPlanet hooks: storm transitions to weather-YYYYMMDD.jsonl, sun angle, the nearest planet''s weather, hazard tables, substances and biome, and the storm countdown built from the game''s storm timing.'
type: snapshot
area: live-hook
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0042]
supersedes: []
superseded_by: []
related_specs: [SPEC-0045, SPEC-0049]
screenshots: []
---

# SPEC-0044: Weather, storms and the current planet's generation data

> Part of the injected `live_position` mod (SPEC-0040). Save before attaching.

## Summary

Three more hooks in `live_position.py` give the map a storm countdown and a description of the planet
the player is on: each storm state change (`cGcSky::SetStormState`), the sun angle (`cGcSky::Update`),
and the current planet's own data (`cGcPlanet::UpdateWeather`): weather type, storm frequency and
intensity, hazard tables, the three substances, life, race, biome, size and terrain colours. Only
values stored inline in those structs are read. The storm countdown is computed by the exporter from
the last transition and the game's storm timing constants, corrected by measurement.

## Inputs and outputs

| Hook | Rate | Writes |
|---|---|---|
| `cGcSky.SetStormState.after(this, leNewState)` | each transition | `weather.storm` in `live.json`; a line in `weather-YYYYMMDD.jsonl` |
| `cGcSky.Update.after(this, lfTimeStep)` | sampled once per 1.0 s | `weather.sun_angle` = `mfSolarSunAngle` (raw, 4 decimals) |
| `cGcPlanet.UpdateWeather.after(this, lfTimeStep)` | runs per planet; kept for the nearest, every `PLANET_INTERVAL = 5.0` s | `weather.planet`, `weather.planet_indices_seen` |

Formats: `docs/hook-events.md` (`live.json` weather block, `weather-*.jsonl`).

## Requirements

### SPEC-0044-REQ-001: Storm transitions
On every `SetStormState` the mod shall record `{"state": <eStormState name>, "t": now}` as the current
storm and append `{"state", "t", "planet_index"}` (planet_index = the last
`cGcPlayerEnvironment.miNearestPlanetIndex`) to `weather-YYYYMMDD.jsonl`. `eStormState`: Inactive 0,
Warning 1, TransitionIn 2, Active 3, TransitionOut 4. Decode the argument with a fallback to the raw
value (never raise). A storm already running when the mod loads is first seen at its next transition.

### SPEC-0044-REQ-002: Planet data
In `UpdateWeather`, the mod shall add `miPlanetIndex` to `planet_indices_seen` and, when it equals
the nearest planet index and the 5 s interval has passed, read `this.contents.mPlanetData`:

| Field | Source (all inline) |
|---|---|
| `planet_index` | `miPlanetIndex` |
| `weather`, `storm_frequency`, `intensity` | `Weather.WeatherType`, `.StormFrequency`, `.WeatherIntensity` (enum names) |
| `hazard` | `Hazard.{Temperature,Toxicity,Radiation,LifeSupportDrain,SpookLevel}`: 6 floats each, 2 decimals |
| `substances` | `CommonSubstanceID`, `UncommonSubstanceID`, `RareSubstanceID` (TkID: bytes up to NUL, leading `^` stripped) |
| `life`, `creature_life`, `race`, `buildings` | `Life`, `CreatureLife`, `InhabitingRace`, `BuildingLevel` |
| `biome`, `biome_subtype`, `planet_class`, `size` | `GenerationData.Biome`, `.BiomeSubType`, `.Class`, `.Size` |
| `tile_colours` | `TileColours`: 23 `cTkColour`, RGB clamped to [0,1], 3 decimals |

### SPEC-0044-REQ-003: Hazard slot order
The system shall label the six values of each hazard table, in order: **Ambient, Water, Cave, Storm,
Night, DeepWater** (from the weather definition files, e.g. humid weather). The live values are the
weather file's Normal or Extreme column for the planet. Measured on an Extreme humid planet:
Temperature 20, 10, 12, 140, 12, -158 (the Extreme column; storms there peak near 140); HUD read
18.6 C ambient at sun angle 117.96, so the HUD applies something more. HUD toxicity 8.9 and radiation
0.9 with zero tables: the HUD does not come from these tables alone.

### SPEC-0044-REQ-004: Storm timing constants
The exporter shall use these seconds (game globals `gcskyglobals`, build 25800958), keyed by
`storm_frequency`, except that **any planet with intensity `Extreme` uses the `Extreme` row**:

| Key | Storm length | Gap between storms |
|---|---|---|
| Low | 120-180 | 900-2400 |
| High | 150-320 | 120-620 |
| Extreme | 150-320 (High length) | 240-620 (`ExtremeFallback`) |

Warning `STORM_WARNING_S = 25`; build-up and clearing `STORM_TRANSITION_S = 20` each. Storm length
excludes the transitions.
Measured, Extreme humid planet with StormFrequency **Low** (weather log timestamps): Warning ->
TransitionIn 25.0 s, TransitionIn -> Active 20.1 s, Active -> TransitionOut **315.0 s**,
TransitionOut -> Inactive 20.0 s; next Active **359.2 s** after Inactive, with no Warning or
TransitionIn logged (player inside the corvette). 315 s is outside Low (120-180) and inside High;
359 s is outside Low's gap (900-2400) and inside ExtremeFallback. Two storms on one planet.

### SPEC-0044-REQ-005: Storm countdown
Given the last transition `{state, t}`, `el = now - t`, and the timing row of REQ-004:

| Condition | Output |
|---|---|
| frequency `Always` | note "constant storms" |
| frequency `None_` | note "no storms on this planet" |
| no transition seen | note "no storm change seen since the mod loaded" |
| Warning | `starts_in = max(0, 25 - el)` |
| TransitionIn | `ends_in = [lo+20-el, hi+20-el]` (each floored at 0) |
| Active | `ends_in = [lo-el, hi-el]` |
| TransitionOut | `clears_in = max(0, 20 - el)` |
| Inactive | `next_in = [gap_lo-el, gap_hi-el]` |

Ranges, not single numbers: the game picks within them.

### SPEC-0044-REQ-006: Substance names
The exporter shall name substance IDs (`YELLOW2`, `TOXIC1`, ...) from the game's item table
(game-data area), falling back to the ID.

## Observed storm log

The first evening's `weather-*.jsonl` on one Extreme humid planet (StormFrequency Low), as deltas
from the previous line. This is the measurement behind REQ-004.

| # | State | Seconds since previous | Note |
|---|---|---|---|
| 1 | Warning | - | |
| 2 | TransitionIn | 25.0 | = warning time |
| 3 | Active | 20.1 | = transition time |
| 4 | TransitionOut | 315.0 | storm length: High range, not Low |
| 5 | Inactive | 20.0 | |
| 6 | Active | 359.2 | gap: ExtremeFallback range; no Warning/TransitionIn (in corvette) |
| 7 | TransitionOut | 251.5 | within High length |
| 8 | Inactive | 18.2 | |
| 9 | Warning | 369.4 | gap again in 240-620 |
| 10 | TransitionIn | 12.6 | shorter than the 25 s warning; not explained |
| 11 | Inactive | 163.2 | no Active logged; the galaxy map was in use at this moment (a system name was generated in the same 0.1 s) |
| 12 | Active | 662.0 | after galaxy-map and system-map use; not a clean gap |
| 13 | Inactive | 111.8 | |

Lines 10-13 overlap galaxy-map and system-map use (seen in `systems-*.jsonl`), so only 1-9 count as
clean measurements. Why a cycle can skip states is not established.

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| No transition since load | `storm: null`; countdown note says so. |
| Player left the planet mid-cycle | The last transition is from the old planet; transitions continue only when near a planet. Show countdown only when the planet matches. |
| Transition skipped (Active without Warning) | Observed once from inside the corvette; countdown jumps to the Active row. |
| `UpdateWeather` not called for the nearest planet (space) | `weather.planet` keeps the last planet; the exporter shows it only when on that planet. |
| Hook disabled | Block freezes; position still updates. Census per SPEC-0041. |

## Verification

1. Wait out a storm on a planet: `weather-*.jsonl` shows Warning, TransitionIn, Active,
   TransitionOut, Inactive with gaps ~25, ~20, length, ~20 s.
2. On a planet with intensity Extreme, the measured Active length falls in 150-320 s.
3. Compare `substances` with the planet's resources in the in-game discovery page; compare
   `weather.planet.weather` with the planet's weather description.
4. On a planet with storm frequency `None_`, no transitions are logged and the note says no storms.

## Lessons

See [SPEC-0049](../lessons/SPEC-0049-live-hook-lessons.md): storms far longer than the Low table.
