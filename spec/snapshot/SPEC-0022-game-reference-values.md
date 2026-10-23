---
id: SPEC-0022
title: 'Game reference values: hotspots, gases, power, storms, crops, refiners'
description: 'The game-data numbers a rebuilder needs to reproduce farm rates, power verdicts and storm countdowns, with the file and field each comes from and what is inferred rather than read.'
type: snapshot
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0020, SPEC-0021]
supersedes: []
superseded_by: []
related_specs: [SPEC-0026, SPEC-0039, SPEC-0060, SPEC-0005, SPEC-0015, SPEC-0043, SPEC-0044, SPEC-0045, SPEC-0063, SPEC-0066, SPEC-0072]
screenshots: [hotspot-class-panel.png, power-check.png, storm-countdown.png]
---

# SPEC-0022: Game reference values

## Summary

The exporter, the map and the farm checks are built on a handful of game constants. This spec
records them as observed on **Steam build 25800958 (Cosmos 7.06)**, with the packed file and field
each was read from, so a rebuilder can (a) check their own extraction produced the same numbers and
(b) know which numbers are game-data, which are measured, and which are guesses. Every value here
can change in a game update; the rebuild shall read them from the installed build (SPEC-0021) and
use this page only as the expected answer for that build.

Coordinate frames of save positions (planet-centred bases, base-local objects, compass north =
planet +Y, nominal radius, sea level) are specified in SPEC-0043 (live-hook) and SPEC-0060 (map
frames); live storm states and the current planet's weather in SPEC-0044. They are not repeated here.

## Inputs and outputs

Inputs: `data/nms_tables.json` (SPEC-0021) and the cited excerpts under
`research/sources/game-data/<build>/` (SPEC-0026). Output: these constants, consumed by the
exporter's farm, power and refiner code and the map's storm countdown.

## Requirements

### SPEC-0022-REQ-001: Hotspot categories and classes

The system shall model six hotspot categories from `regionhotspotstable` `RegionHotspots`, each
with `ProbabilityWeighting` 1.0:

| Category | MinRange–MaxRange (m) | Class weighting C:B:A:S | Class strength C/B/A/S |
|---|---|---|---|
| Power | 200–250 | 10:6:2:1 | 150 / 220 / 250 / 300 (kP) |
| Mineral1, Mineral2, Mineral3 | 190–225 | 6:4:2:1 | 1 / 1.5 / 2 / 2.5 (multiplier) |
| Gas1, Gas2 | 190–225 | 20:4:2:1 | 1 / 1.5 / 2 / 2.5 (multiplier) |

Table-level fields: `RegionHotspotsPoleSpacing` 900, `RegionHotspotsPerPoleMin` 1.9,
`RegionHotspotsPerPoleMax` 3.8, `RegionHotspotsMinSameCategorySpacing` 10,
`RegionHotspotsMaxDifferentCategoryOverlap` 150; Power has `DiscoveryDistanceThreshold` 0.95.
Survey device range `SurveyMaxDistance` 400 m (gameplay globals). Falloff
(`gcenvironmentglobals`): `MinHotspotFalloffDistance` 100, `MaxHotspotFalloffDistance` 500,
`MaxHotspotOffsetDistance` 150. Current belief (not yet measured): full yield within 100 m of the
centre, zero at 500 m, and the survey marker can sit up to 150 m off the true centre.

### SPEC-0022-REQ-002: Extraction rates

The system shall compute a single extractor's rate as
`100 units/s x 3600 / 1440 units per item x class strength` =
**250 / 375 / 500 / 625 items per hour** for class C / B / A / S, at the hotspot centre and before
network diminishing returns. An extractor stores 250 items (360,000 units); a supply depot 1,000
(1,440,000). Mineral (`U_EXTRACTOR_S`) and gas (`U_GASEXTRACTOR`) extractors share these numbers.
The save stores each extractor's and depot's fill as `UserData >> 32` in link-grid units (SPEC-0005).

Network diminishing returns are **not in the data** (believed hardcoded). Measured cross-section:
networks of 2–12 extractors on different hotspots all measured 870–1,570 items/h per network;
one standalone extractor measured 300/h dropping to 200/h once piped into a network. The rebuild
shall not invent a formula; it shall measure network rates from consecutive saves.

### SPEC-0022-REQ-003: Hotspot class is not observable from data

The system shall not attempt to derive a hotspot's class from the save or from the game's hotspot
marker: the save stores nothing about hotspots, and a class B and a class S marker differ only in
position, label and marker type. The class is read by the player from the visor and entered by
hand (SPEC-0063, SPEC-0072).

### SPEC-0022-REQ-004: Biome gases

The system shall map Gas1 by biome from `RegionHotspotBiomeGases`; Gas2 is always Oxygen.

| Gas1 | Id | Biomes |
|---|---|---|
| Nitrogen | GAS3 | Lush, Toxic, Blue, Swamp |
| Radon | GAS2 | Radioactive, Frozen, Green |
| Sulphurine | GAS1 | Scorched, Barren, Red, Lava |
| Methane | GAS4 | Waterworld, GasGiant |
| Oxygen | OXYGEN | Dead, Weird |

Observed live: a Toxic planet's class S gas hotspot gave Nitrogen.

### SPEC-0022-REQ-005: Extractable substances

The system shall treat `RegionHotspotSubstances` (33 entries, each `AmountCost` 360,000 for
`SubstanceYeild` 250) as the set a hotspot *can* yield: GAS1–GAS4, OXYGEN; YELLOW2 Copper,
RED2 Cadmium, GREEN2 Emeril, BLUE2 Indium, PURPLE2 Quartzite and their activated forms
EX_YELLOW/EX_RED/EX_GREEN/EX_BLUE/EX_PURPLE; DUSTY1 Pyrite, LUSH1 Paraffinium, TOXIC1 Ammonia,
RADIO1 Uranium, COLD1 Dioxite, HOT1 Phosphorus, WATER1 Salt, CAVE1 Cobalt, ASTEROID1 Silver,
LAND3 Magnetised Ferrite, SPACEGUNK3 Rusted Metal, ASTEROID2 Gold, CATALYST1 Sodium, LAVA1 Basalt,
PLANT_POOP Faecium, CREATURE1 Mordite, GASGIANT1 Crystallised Helium, WATERWORLD1 Lithium.
Which of them a given planet's mineral hotspots give is not in the table: observed live, a mineral
hotspot can give any of the planet's three listed resources (common on one planet, rare on
another), and the marker does not say which.

### SPEC-0022-REQ-006: Base power

The system shall use these link-grid ratings (`basebuildingobjectstable` `LinkGridData`, Power
network, units kP):

| Part | Rating |
|---|---|
| `U_EXTRACTOR_S`, `U_GASEXTRACTOR` | depends on Power **-50**, effect `EnablesRate` |
| `U_GENERATOR_S` (electromagnetic) | rate 1 x its Power hotspot's class strength = **150 / 220 / 250 / 300** for C/B/A/S |
| `U_SOLAR_S` | **+50**, `DependsOnEnvironment` DayNight (by day only) |
| `U_BIOGENERATOR` | Fuel network -1/s, storage 180,000; dependent Power **+50** while fuelled |
| `U_BATTERY_S` | storage **45,000** |
| Teleporter | -20; containers -5; planters and the bio-dome -50; rooms, corridors, lights, doors also draw |

Battery charge in the save is `UserData >> 32` (13 of 50 batteries read exactly 45,000 on one
save). The unit is assumed to be kP-seconds (45,000 = 15 minutes of one extractor); not measured.

The save does **not** hold: the class of the power hotspot under each electromagnetic generator;
biofuel fuel (every biofuel generator has `UserData` 256); a traceable wire graph (saved wires stop
4–15 m short of generators). Rooms and lights draw only when wired, so a per-base power verdict
shall count extractors as the demand and report the rest as "up to".

### SPEC-0022-REQ-007: Storm timing

The system shall predict storms from `gcskyglobals.globals`:

| Field | Low | High | Extreme fallback |
|---|---|---|---|
| Storm length (s), min–max | 120–180 | 150–320 | — |
| Time between storms (s), min–max | 900–2400 | 120–620 | 240–620 |

`StormWarningTime` 25 s; `StormTransitionTime` 20 s (build-up and clearing each). Storm states
(from the running game): Inactive, Warning, TransitionIn, Active, TransitionOut.

Measured once, on an Extreme humid planet whose StormFrequency was Low: warning 25.0 s, build-up
20.1 s, storm 315.0 s, clearing 20.0 s, next storm 359 s after the first cleared. 315 s fits only
the High length and 359 s fits only the Extreme fallback gap, so **an Extreme planet shall be
treated as High length with the ExtremeFallback gap whatever its StormFrequency** (two storms of
evidence; flag it as provisional). A storm can reach Active without Warning being reported (seen
once, from inside a vehicle).

### SPEC-0022-REQ-008: Hazard table slots

The system shall index a planet's hazard arrays (`cGcPlanetHazardData`: Temperature, Toxicity,
Radiation, LifeSupportDrain, SpookLevel) as six slots in the order **Ambient, Water, Cave, Storm,
Night, DeepWater**. Each weather file (`metadata/simulation/solarsystem/weather/<kind>weather`)
gives a Normal and an Extreme value per slot (humid: ambient 12/20, storm 92/140, deep water
-95/-158). The HUD's readout is not the table value alone (18.6 C shown against a table 20).

### SPEC-0022-REQ-009: Crops

The system shall use, per planted crop (`rewardtable` `PLANT_*`, grow time = the part's
PlantGrowth storage):

| Crop | Yields | Per harvest | Grow (h) |
|---|---|---|---|
| Frostwort | Frost Crystal | 50 | 1 |
| Gamma Weed | Gamma Root | 50 | 4 |
| Fungal Cluster | Fungal Mould | 50 | 4 |
| Star Bramble | Star Bulb | 25 | 4 |
| Gutrot Flower | Faecium | 25 | 4 |
| NipNip | NipNip Buds | 1 | 4 |
| Venom Urchin | Sac Venom | 1 | 3.33 |
| Gravitino Host | Gravitino Ball | 1 | 2 |
| Albumen Pearl Orb | Albumen Pearl | 1 | 2 |
| Mordite Root | Mordite | 25 | 8 |
| Solar Vine | Solanium | 50 | 16 |
| Echinocactus | Cactus Flesh | 100 | 16 |

Harvest All Plants has a radius of 6.0 in the freighter Hydroponic Room and the Biodome
(SPEC-0025).

### SPEC-0022-REQ-010: Refiners and economy

Refiner seconds per operation = `TimeToMake / 250` for substance outputs, `/ 2` for product
outputs (survival mode 100 / 1). A refiner's output slot caps at the item's stack size: 9,999 for
substances but 10 for products, so a product recipe stalls quickly. Selling price = `BaseValue` x
the system's modifier between `1 + LowPriceMod` and `1 + HighPriceMod`; the per-sale price drop is
in code, not data, scaled by `EconomyInfluenceMultiplier`. Frigate expedition length = events x
5,400 s. Units are a 32-bit value written signed: a negative save value plus 2^32 is the balance.

### SPEC-0022-REQ-011: Biome odds per star

The system shall take biome availability from `biomelistperstartype`: a weight of 0 means never.
Notably Waterworld rolls only under purple stars (weight 0.66), Swamp and Lava only under purple
(0.2), and GasGiant has weight 0 everywhere in that table.

## Degraded behaviour

- A different build may change any number above. The rebuild shall prefer its own extracted
  values and use this table only to confirm extraction on build 25800958.
- Where a value is marked measured or provisional (storms on Extreme planets, battery unit,
  network falloff), the UI shall present derived predictions as estimates.
- If `nms_tables.json` lacks a section, the dependent panel shall say what is missing rather than
  fall back to these constants silently.

## Verification

- Extract tables (SPEC-0021) on build 25800958 and compare every value in REQ-001–REQ-010; all
  shall match exactly.
- `nms-pak get regionhotspotstable` and read the `Power` block: weightings 10/6/2/1, strengths
  150/220/250/300.
- A real farm: one unpiped extractor on a known class-C hotspot centre, with power, shall fill at
  about 250 items/h between two saves an hour apart.
- Storm countdown: on a Low-frequency non-extreme planet, consecutive storms shall be 900–2,400 s
  apart.

## Lessons

See [SPEC-0039](../lessons/SPEC-0039-game-data-lessons.md): **data is not behaviour** (a table
value such as Harvest All's 6.0 or a storm range says what the game is configured with, not what
the player sees; the Extreme storm planet broke the obvious reading), and **the save omits what you
most want** (hotspot class, generator hotspot, fuel) so the UI has to ask the player.
