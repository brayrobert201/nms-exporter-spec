---
id: SPEC-0007
title: 'Player sections of /json: inventory, plants, fleet, missions, technology, glyphs, tidy'
description: 'How each remaining section of the snapshot is read from PlayerStateData: containers and item totals, plants, frigates and expeditions, missions against the game tables, the technology placement review, portal glyphs, lifetime counters and the storage tidy plan.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0001, SPEC-0002]
supersedes: []
superseded_by: []
related_specs: [SPEC-0003, SPEC-0006, SPEC-0009, SPEC-0021, SPEC-0023]
screenshots: [conky-traveller.png, conky-bases.png]
---

# SPEC-0007: Player sections of /json

## Summary

Beyond refiners and farms, the snapshot carries everything the wall screens and dashboards show
about the player: what is in every container, what is planted, the frigate fleet, mission
progress, installed technology with a placement review, where the player is as portal glyphs, the
lifetime counters, and a storage tidy plan. Each is a pure function of `PlayerStateData` plus the
game tables. All paths below are relative to `ps = BaseContext.PlayerStateData`, ids with `^`
stripped.

## Inputs and outputs

Tables used: `items[id] {name, type, category, value, craft, stat, class, procedural, core,
upgrade}`, `missions[id] {title, final, stages [[num, text]], rewards [[id, [desc]]], leads_to}`,
`fleet {event_seconds: 5400, stat_points_per_level: 5, level_thresholds_community:
[4,8,15,25,30,35,40,45,50,55], traits{id: {name, stat, points}}}`, `buildings[id].storage` and
`.network == "PlantGrowth"` (grow seconds). Optional `data/nms_storage_plan.json`.

## Requirements

### SPEC-0007-REQ-001: Item names

`name(id)`: for a procedural id `<type>#<seed>`, `"<name of type> #<seed>"` when the type is
printable ASCII and in the tables, else `"Procedural item #<seed>"`; otherwise the table name or
the raw id. Base-building parts, corvette parts and procedural items live in three product
tables of their own, which the tables extract must include.

### SPEC-0007-REQ-002: Containers

Inventories are read from these `ps` keys, with these labels: `Inventory` Exosuit,
`Inventory_Cargo` Exosuit cargo, `FreighterInventory` Freighter, `FreighterInventory_Cargo`
Freighter cargo, `ChestMagicInventory` / `ChestMagic2Inventory` Storage (magic / magic 2),
`CookingIngredientsInventory`, `CorvetteStorageInventory`, `FishPlatformInventory`,
`FishBaitBoxInventory`, `FoodUnitInventory`, `Chest1Inventory`...`Chest10Inventory` Storage 1-10;
plus each `ShipOwnership[i].Inventory` / `.Inventory_Cargo` ("Ship n: <name>", "(cargo)") and
each `VehicleOwnership[i].Inventory` ("Exocraft n: <name>"). Only slots with
`Type.InventoryType` in `Product`/`Substance` count. Outputs: `locations[{location, id, item,
type, amount}]` and `items{id: {item, type, total, stacks, locations}}`.

### SPEC-0007-REQ-003: Player, universe, counters

- `player`: units/nanites/quicksilver (unsigned, SPEC-0001-REQ-009), `Health`, `Shield`,
  `Energy` (life support), `ShipHealth`, `TimeAlive`, `CommonStateData.TotalPlayTime`, ammo
  (`BoltAmmo`, `ScatterAmmo`, `PulseAmmo`, `LaserAmmo`), `Hazard` list.
- `universe` from `UniverseAddress`: `galaxy = RealityIndex`, `GalacticAddress.{VoxelX, VoxelY,
  VoxelZ, SolarSystemIndex, PlanetIndex}`, plus
  `glyphs = "%X%03X%02X%03X%03X" % (planet & 0xF, system & 0xFFF, y & 0xFF, z & 0xFFF, x & 0xFFF)`
  and the signal-booster address
  `"%04X:%04X:%04X:%04X" % (x + 2047, y + 127, z + 2047, system)`.
- `stats`: every entry of the `Stats` group with `GroupId == "^GLOBAL_STATS"`, value
  `IntValue` or `FloatValue` (an empty dict is 0).
- `collections`: ships (named or with slots), multitools, pets, eggs, frigates, bases,
  visited systems, known tech/products/recipes/words, discoveries, vehicles, named settlements.

### SPEC-0007-REQ-004: Plants

A base object is a plant when its id's building record has `network == "PlantGrowth"` and a
storage (= grow seconds). Per plant: `plant` (display name), `id`, `yield_id`, `yields`,
`grown = UserData >> 32` (seconds grown), `grow_seconds`, `anchor = Timestamp`. Display names and
yields for the 12 farmable plants:

| Part id | Plant | Yields |
|---|---|---|
| RADIOPLANT | Gamma Weed | PLANT_RADIO (Gamma Root) |
| TOXICPLANT | Fungal Cluster | PLANT_TOXIC (Fungal Mould) |
| SNOWPLANT | Frostwort | PLANT_SNOW (Frost Crystal) |
| SACVENOMPLANT | Venom Urchin | SACVENOM |
| SCORCHEDPLANT | Solar Vine | PLANT_HOT (Solanium) |
| POOPPLANT | Coprite Sphere | PLANT_POOP (Faecium) |
| GRAVPLANT | Gravitino Host | GRAVBALL |
| CREATUREPLANT | Mordite Root | CREATURE1 (Mordite) |
| BARRENPLANT | Echinocactus | PLANT_DUST (Cactus Flesh) |
| LUSHPLANT | Star Bramble | PLANT_LUSH (Star Bulb) |
| PEARLPLANT | Albumen Pearl Orb | ALBUMENPEARL |
| NIPPLANT | NipNip Buds | NIPNIPBUDS |

Metrics also group by (base, kind) with mature count, next-ready seconds and the stock of the
yield.

### SPEC-0007-REQ-005: Fleet

Expeditions from `FleetExpeditions[]`: category, duration class, `StartTime`, events (`Events`
length), `NextEventToTrigger` (done), success/fail counts, active/damaged/destroyed frigate
counts, `eta = start + events * 5400`. Frigate classes counted from `FleetFrigates[].FrigateClass`.
Roster per frigate: stats `[combat, exploration, industrial, trade, fuel_burn, fuel_capacity,
speed]` from `Stats[0..6]`, traits from `TraitIDs`, `expeditions = TotalNumberOfExpeditions`,
`level = count of thresholds < expeditions`, and the **base-roll score**:

```
main_total = combat + exploration + industrial + trade
roll       = main_total - Σ points of traits whose stat is Combat/Exploration/Mining/Diplomatic
                        - 5 * level
score      = clamp(roll, -5, 14)          # the community frigate calculator's scale
```

### SPEC-0007-REQ-006: Missions

For each `MissionProgress[]` entry, skip ids matching
`_HINT$|_WIKI$|_LORE$|_LORE_$|_DUMMY$|_FIX$|_FIX_\d$|_CLEAN$|HELP$|^DM_|^D_NEXUS|^WM_|^COMM_|^S\d\d_|CATCHUP`.
Status: `unknown` (not in tables), `not_started` (`Progress == -1`), `complete`
(`Progress == 2147483647` or `>= final`), else `in_progress`. Seeded missions repeat; an
in-progress instance wins. `Progress` is a progress **number**, not a stage index: the current
objective is the text of the last stage whose number is `<= Progress`. Rewards are the mission's
own reward entries (what finishing the whole mission pays), first 4. Sort in-progress by
`final - progress` ascending ("closest to done first"). `selected` marks `CurrentMissionID`.

### SPEC-0007-REQ-007: Technology placement review

Per tech inventory (`Inventory_TechOnly` exosuit, `Inventory` exosuit general, each
`Multitools[i].Store`, each `ShipOwnership[i].Inventory_TechOnly`,
`FreighterInventory_TechOnly`, each `VehicleOwnership[i].Inventory_TechOnly` with vehicle index
names Roamer, Nomad, Colossus, Pilgrim, (unused), Nautilon, Minotaur):

- slots of type `Technology` with `x, y`, `family` (the table's stat family; a procedural
  module's template, e.g. `T_JET`, is itself a tech entry whose `BaseStat` names it), class
  (table, else the id's trailing digit 1-4 -> C/B/A/S or X), procedural/core/upgrade flags,
  broken (`!FullyInstalled` or `DamageFactor > 0`);
- supercharged slots = `SpecialSlots` with type `TechBonus`; each is good (procedural module),
  weak (fixed technology) or empty;
- `move_in`: procedural modules on plain slots, best class first, up to weak + empty count;
  `upgrade`: better modules that could displace the weakest supercharged one (top 3);
- `isolated`: in families holding a module, modules and core techs with no family member at
  Manhattan distance 1.

Module rolled stats are not in the save, so the review is by class and family only.

### SPEC-0007-REQ-008: Storage tidy plan

Where `data/nms_storage_plan.json` loads (`{tidy: [container labels], default, containers:
[{container, label, ids, patterns, categories}]}`), each stack in the `tidy` containers gets a
target: first container whose `ids` or regex `patterns` match the base id (in list order), else
first whose `categories` contain the item's table category, else `default`; a target outside
`tidy` means it stays. Output: `moves[{id, item, amount, from, to, to_label}]`, per container
`{stacks_now, stacks_after = Σ ceil(amount/max) per item, capacity (ValidSlotIndices count or
W×H), incoming, outgoing, merges, over_capacity}`. Containers not under `tidy` are never asked to
give anything up. The file is re-read on every save.

### SPEC-0007-REQ-009: Blueprints and raw positions

`blueprints{id: {item, yield, ingredients}}` for every known product with a blueprint (used by
the screen's recipe-aware search). The builder also keeps `raw_bases` (SPEC-0002 `/raw/bases`)
and `raw_points` (vehicle positions, drained deposits, settlements, `KnownPortalRunes` bitmask)
for the `map-ui` area.

## Degraded behaviour

Each section reads with `.get()` and defaults, so a missing key yields an empty section, not an
exception. Missing mission tables: `missions.tables = false`, all `unknown`. Missing storage
plan: `tidy = {enabled: false, error}` and no tidy metrics. Unknown item ids show raw.

## Verification

- Inventory search on the bases screen for an item: the per-container amounts match the game.
- Traveller screen glyphs match the portal address shown in game for the current system.
- A frigate's score matches the community frigate calculator for the same stats and traits.
- The current mission's objective text matches the in-game log.

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): mission `Progress` is a number per
mission version, not a stage index; `VehicleOwnership` is indexed by vehicle type.
