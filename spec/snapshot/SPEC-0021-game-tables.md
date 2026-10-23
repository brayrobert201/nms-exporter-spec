---
id: SPEC-0021
title: 'Game tables: building nms_tables.json from the installed build'
description: 'Extract recipes, items, prices, hotspot and extractor constants, base-part link-grid rates, crops, biomes, fleet timings and mission tables from the game paks into one JSON file the exporter reads at runtime.'
type: snapshot
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0020]
supersedes: []
superseded_by: []
related_specs: [SPEC-0022, SPEC-0030, SPEC-0032, SPEC-0039, SPEC-0004, SPEC-0005, SPEC-0006, SPEC-0007]
screenshots: []
---

# SPEC-0021: Game tables

## Summary

The exporter (SPEC-0002; refiners SPEC-0004, farms SPEC-0005, planner SPEC-0006, missions SPEC-0007) turns a save into farm rates, refiner ETAs,
a crafting planner, mission objectives and power verdicts. Every one of those needs a number that
is not in the save: how long a recipe takes, how many link-grid units make one item, what a
class-B hotspot multiplies by, what mission progress value means "done". The extractor reads those
numbers out of the installed game's own tables once per game update and writes them to
`data/nms_tables.json`. Only the JSON is used at runtime; the extractor is a maintenance tool.
Numbers come from the installed build, not a wiki (SPEC-0030).

## Inputs and outputs

**Inputs:** the paks `NMSARC.MetadataEtc.pak`, `NMSARC.globals.pak`, `NMSARC.misc.pak`,
`NMSARC.Precache.pak`; HGPAKtool and MBINCompiler (SPEC-0020); the Steam manifest for the build
id. CLI: `--pcbanks DIR --work DIR --out FILE --hgpaktool BIN --mbincompiler BIN`, all with
defaults (work dir `<tmp>/nmsdata`, out `data/nms_tables.json`).

**Unpack filters** (passed as repeated `hgpaktool -U -O <work> -f <glob>`, all four paks at once):

```
*GCRECIPETABLE*  *GCSUBSTANCETABLE*  *GCPRODUCTTABLE*  *GCTECHNOLOGYTABLE*
LANGUAGE/NMS_*_ENGLISH.MBIN  *GCGAMEPLAYGLOBALS*  *GCENVIRONMENTGLOBALS*
*REGIONHOTSPOTSTABLE*  *BASEBUILDINGOBJECTSTABLE*  *GCFLEETGLOBALS*  *BIOMELISTPERSTARTYPE*
*NMS_BASEPARTPRODUCTS*  *NMS_MODULARCUSTOMISATIONPRODUCTS*  *PROCEDURALPRODUCTTABLE*
*MISSIONS/TABLES*  *REALITY/TABLES/REWARDTABLE*  *PROCEDURALTECHNOLOGYTABLE*  *FRIGATETRAITTABLE*
```

Exact packed paths: [docs/game-data-locations.md](../../docs/game-data-locations.md).

**Output:** one JSON object, indent 1, with top-level keys in this order:
`meta, units, refiner, extraction, fleet, buildings, crops, planets, items, recipes, missions`.
Build 25800958 gives about 2.5 MB: 1,684 recipes (361 refiner, 1,323 cooking), 5,235 items
(114 substances, 4,474 products, 647 technologies), 101 rated base parts, 12 crops, 499 missions.

## Requirements

### SPEC-0021-REQ-001: Unpack and convert incrementally

When run, the extractor shall unpack with the filters above (always; it takes under a second and
keeps the MBINs current), then convert with MBINCompiler only those `.mbin` files (excluding any
path containing `usenglish`) whose `.MXML` is missing or older than the MBIN. `DOTNET_ROOT`
defaults to `<tools>/dotnet8`. Any failing external command aborts with its stdout and stderr.

### SPEC-0021-REQ-002: MXML reading conventions

The extractor shall parse MXML with a standard XML parser. In MXML every field is
`<Property name="Field" value="...">`; a struct-valued field has `value="<TypeName>"` and child
Properties; list elements are child Properties; enum-wrapped fields (e.g. `Type`) carry the value
in a child Property named for the enum (`InventoryType`, `LinkNetworkType`, `FrigateStatType`,
`FrigateTraitStrength`). Numbers are written `1.000000`; store them as int when integral.
Files are located by **basename regex**, case-insensitive, over the work directory.

### SPEC-0021-REQ-003: Localisation

The extractor shall build `loc = {Id: English}` from every `TkLocalisationEntry` in every
`nms_*_english.MXML`. Names everywhere below resolve through `loc`, falling back to the key.

### SPEC-0021-REQ-004: Items

The extractor shall emit `items[<ID>]` from these tables and element types:

| Table (basename regex) | Element | `type` |
|---|---|---|
| `gcsubstancetable\.mxml$` | `GcRealitySubstanceData` | Substance |
| `gcproducttable\.mxml$`, `^nms_basepartproducts\.mxml$`, `^nms_modularcustomisationproducts\.mxml$`, `^proceduralproducttable\.mxml$` | `GcProductData` | Product |
| `gctechnologytable\.mxml$` | `GcTechnology` | Technology |

Fields: `name` = loc(`NameLower` or `Name`); if that is empty, contains `%`, or did not resolve,
use loc(`Subtitle`) (procedural name templates). `category` = substance `Category` or product
`Type` (first child's value). `symbol` = loc(`Symbol`) if present. `value` = `BaseValue`
(not technology). `craft` = `{ingredients: [{id, type, amount}], yield: CraftAmountMultiplier or 1}`
when a product has `IsCraftable` true and non-empty `Requirements`. `price` for products =
`{low: 1+LowPriceMod, high: 1+HighPriceMod, station_markup: SpaceStationMarkup, influence:
EconomyInfluenceMultiplier}` (4 d.p.). Technology adds `stat` (`BaseStat`), `category`, `core`,
`upgrade` booleans.

Procedural upgrade modules (`gcproceduraltechnologytable`, `GcProceduralTechnologyData`) shall be
added as `{name: "<base> module", type: Technology, procedural: true, template, quality, class,
sentinel, stat, category, core: false, upgrade: true}`, where `stat`/`category` come from the
`Template` technology and class maps Quality: Normal C, Rare B, Epic A, Legendary S, Illegal X,
Sentinel S, SeaTrash C.

### SPEC-0021-REQ-005: Refiner speed and recipes

The extractor shall read from `gcgameplayglobals.global`: `RefinerSubsMadeInTime` (250),
`RefinerProductsMadeInTime` (2), `RefinerSubsMadeInTimeSurvival` (100),
`RefinerProductsMadeInTimeSurvival` (1) into `refiner.made_in_time`.

For each entry of `gcrecipetable`'s `Table` it shall emit `{id, operation, name, cooking,
time_to_make, seconds_per_op, seconds_per_op_survival, result{id,type,amount},
ingredients[{id,type,amount}]}` where `operation` = loc(`RecipeType`) with a leading
`Requested Operation: ` removed, and `seconds_per_op = TimeToMake / divisor`, divisor 250 when the
result is a Substance and 2 when a Product (survival divisors 100 / 1), rounded to 6 d.p.
One op consumes `ingredients` and yields `result.amount`.

### SPEC-0021-REQ-006: Hotspots and extraction

From `regionhotspotstable` the extractor shall read:

- `RegionHotspots`: per category (`Power`, `Mineral1`–`3`, `Gas1`–`2`) the `ClassWeightings`
  and `ClassStrengths` maps (C/B/A/S) and `range_m = [MinRange, MaxRange]`.
- `RegionHotspotBiomeGases`: per biome `{gas1: Gas1Id, gas2: Gas2Id}`, skipping empty Gas1.
- `RegionHotspotSubstances`: `[{id: SubstanceId, amount_cost: AmountCost, yield: SubstanceYeild}]`
  (sic: the game spells it `Yeild`).

From `gcenvironmentglobals.global`: `MaxHotspotOffsetDistance`, `Min/MaxHotspotFalloffDistance`;
from gameplay globals `SurveyMaxDistance`.

It shall then derive `extraction`: `resource_units_per_item = amount_cost // yield` of the first
substance (360000 / 250 = **1440**); from building `U_EXTRACTOR_S`: `extractor_rate_per_s`,
`extractor_storage_items = storage // 1440`, `power_required_kp = -(first dependency rate)`;
`supply_depot_storage_items` from `U_SILO_S`; `items_per_hour_at_strength_1 =
rate * 3600 / 1440`; `items_per_hour_by_class` = that times `Mineral1` class strength.

### SPEC-0021-REQ-007: Base-part link grid

For every `GcBaseBuildingEntry` in `basebuildingobjectstable` with a `LinkGridData` whose `Rate`
or `Storage` is non-zero or whose `DependsOnHotspots` / `DependsOnEnvironment` is not `None`, the
extractor shall emit `buildings[<ID>] = {network, rate_per_s, storage, depends_on_hotspot,
depends_on_environment, dependencies: [{network, rate: DependentRate, effect: DependentEffect}]}`
with `network` from `Connection.Network.LinkNetworkType`. Power units are kP; Resources storage is
link-grid units (divide by 1440); PlantGrowth storage is seconds to mature.

### SPEC-0021-REQ-008: Crops

The extractor shall map planted-crop base parts to reward entries with this fixed table, read
`AmountMin`/`AmountMax` and the first `ID` from the matching `GcGenericRewardTableEntry` in
`rewardtable`, and emit `crops[<part>] = {plant, yield_id, yields, amount: max, amount_min,
grow_seconds: buildings[part].storage}`:

```
RADIOPLANT PLANT_RADIO   TOXICPLANT PLANT_TOXIC   SNOWPLANT PLANT_SNOW
SACVENOMPLANT PLANT_SACVENOM   SCORCHEDPLANT PLANT_SCORCHED   POOPPLANT PLANT_POOP
GRAVPLANT PLANT_GRAV   CREATUREPLANT PLANT_CREATURE   BARRENPLANT PLANT_BARREN
LUSHPLANT PLANT_LUSH   PEARLPLANT PLANT_PEARL   NIPPLANT PLANT_NIP
```

### SPEC-0021-REQ-009: Planets

The extractor shall emit `planets.biomes_by_star[<star colour>] = {<biome>: weight}` from
`biomelistperstartype` (`StarType` > `BiomeProbability`), and `planets.labels[<biome>]` = sorted
unique loc texts whose key matches `^([A-Z]+?)(\d+)$` with a prefix in
`LUSH Lush, TOXIC Toxic, SCORCHED Scorched, FROZEN Frozen, IRRADIATED Radioactive, BARREN Barren,
DEAD Dead, SWAMPBIOME Swamp, LAVABIOME Lava, WATERWORLD Waterworld, GASGIANT GasGiant,
REDBIOME Red, GREENBIOME Green, BLUEBIOME Blue, GLITCHBIOME Weird`, keeping texts that contain
`%PLANETCLASS%` (replaced by `Planet`) or are under 40 characters.

### SPEC-0021-REQ-010: Fleet

From `gcfleetglobals.global`: `event_seconds` (`TimeTakenForExpeditionEvent`, 5400),
`event_seconds_easy` (900), `events_by_duration` (`ExpeditionDurations.Duration`: VeryShort 4,
Short 6, Medium 10, Long 14, VeryLong 18), `passive_income_tick_seconds`,
`stat_points_per_level`, `trait_points` (`FrigateTraitStrengths.FrigateStatType.*.StatAlteration`);
from `frigatetraittable`, `traits[ID] = {name, stat, strength, points}`. Rank-up thresholds are not
in the data; the extractor writes the community list `[4,8,15,25,30,35,40,45,50,55]` under
`level_thresholds_community` so its provenance is visible in the key name.

### SPEC-0021-REQ-011: Missions

The extractor shall read `GcGenericMissionSequence` from the mission tables in this priority order
(first definition of a `MissionID` wins): `coremissiontable, missiontable, atlaspathtable,
npcmissiontable, sentinelsettlementmissiontable, watermissiontable, tutorialmissiontable,
recurringmissiontable, communitymissiontable, npcbuildersmissiontable, spacepoimissiontable,
basecomputermissiontable, piratemissiontable, fleetmissiontable, corvettemissiontable,
swarmmissiontable, planetprocmissiontable, spaceoutpostmissiontable, startedonusemissiontable,
wikimissiontable, statstoriesmissiontable`. Missions with no resolvable title are dropped.

Per mission: `title` = cleaned loc(`MissionTitles.Format`); `final` = the `Progress` of the
highest `Version` in `FinalStageVersions`; `stages` = `[progress, objective]` pairs collected by
walking `Stages` recursively (a stage with sub-`Stages` passes its objective down; a leaf with
`Versions` contributes `(Progress at max Version, objective or loc(Message))`), sorted and with
consecutive duplicate texts collapsed; `rewards` = `[reward Id, [descriptions]]`; `leads_to` =
values of reward fields named `Mission`; `class`; `recurring`.

Text cleaning: drop `<TAG>` and `<>` markup, keep the first line, replace `%SETTLEMENT%` with
"your settlement" and drop other `%PLACEHOLDERS%`, collapse whitespace, strip a trailing colon.

Reward descriptions shall read each `GcRewardTableItem`'s `Reward` (kind = its first child's name
minus the `GcReward` prefix), skip these 33 bookkeeping kinds:

```
MissionMessage ScanEvent ModifyStat IncrementStat InterventionResponse SetInteractionMissionState
OpenPage WikiTopic RefreshHazProt Health Shield OpenUnlockTree RecycleSpecificObject
CommunityContribution SettlementParty SettlementStat WantedLevel SettlementCustomJudgement
AtlasPathProgress MissionSeeded Trigger SetMissionStat ActivateFiends DamageTech
AdvancePortalState SendChatMessage TriggerMaintenance FrigateFlyby Worker Mission
ExchangeProduct UnlockSeasonReward TeachSpecificWords
```

and render the rest as short phrases from the ids in fields `ID`, `TechId`, `ProductIds`,
`TechIds` (at most 12, unique, in order) and `AmountMin`/`AmountMax`/`Amount`:
`SpecificProduct`/`SpecificSubstance`/`Product` -> `"<amt> x <names>"`; `SpecificTech` ->
`"<names> (tech)"`; recipe kinds -> `"blueprint: <names>"`; `Money` -> `"<amt> units"`;
`ProcTechProduct` -> `"<group> module"`; `TeachWord` -> `"<n> words"`; `DiscoverRune` ->
`"portal glyph"`; `SpecificShip` / `SpecificFrigate` / `SpecificWeapon` / `SpecificPetEgg` /
`UnlockTitle` / `Standing` -> fixed phrases; anything else -> the kind name.

The save's `MissionProgress.Progress` is a progress number, not a stage index: the current
objective is the last stage whose number is at or below it (SPEC-0007).

### SPEC-0021-REQ-012: Self-description

The output shall carry `meta = {generated (UTC ISO), game_build_id, mbincompiler_version (parsed
from the MXML header comment "MBINCompiler version (x.y.z)"), save_key_mapping_url, source}` and a
`units` map of human-readable notes stating the unit and derivation of every non-obvious field
(e.g. `"buildings.storage": "link-grid units; for Resources divide by resource_units_per_item
... for Power it is kPs (battery 45000); for PlantGrowth it is seconds to mature"`).

## Degraded behaviour

- **A required table missing** (renamed in an update): the extractor exits naming the regex
  (`missing <pattern> under <work>`); no partial file is written, so the previous
  `nms_tables.json` stays in place and the exporter keeps working on last build's numbers.
- **A single mission table missing:** skipped silently; missions from other tables still load.
- **MBINCompiler too old for the build:** conversion fails and the run aborts. Get the release
  matching the game version.
- **At runtime**, the exporter shall treat `nms_tables.json` as optional per section: a missing
  section blanks the panels that need it and says why, rather than rendering zeroes (SPEC-0002).
- **Stale tables after an update:** `meta.game_build_id` differs from the installed build; the
  compendium lint (SPEC-0026) and the exporter can warn on that comparison.

## Verification

On build 25800958 (Cosmos 7.06, MBINCompiler 7.04.1.3) the run prints
`wrote data/nms_tables.json: 1684 recipes (361 refiner, 1323 cooking), 5235 items, 101 rated parts,
499 missions`, and:

| Path | Value |
|---|---|
| `refiner.made_in_time` | `{substance: 250, product: 2, survival_substance: 100, survival_product: 1}` |
| `extraction.resource_units_per_item` | 1440 |
| `extraction.extractor_rate_per_s` / `extractor_storage_items` / `supply_depot_storage_items` | 100 / 250 / 1000 |
| `extraction.items_per_hour_by_class` | C 250, B 375, A 500, S 625 |
| `extraction.power_required_kp` | 50 |
| `extraction.survey_max_distance_m` | 400 |
| `extraction.hotspot_falloff` | offset 150, min falloff 100, max falloff 500 |
| `buildings.U_BATTERY_S.storage` | 45000 |
| `crops.SNOWPLANT` | Frostwort -> Frost Crystal, 50, 3600 s |
| `recipes[0]` | `RECIPE_1`, Fermentation, cooking, TimeToMake 5, seconds_per_op 2.5 |
| `fleet.event_seconds` | 5400 |

`scripts/nms-data item Methane`, `crops`, `biomes` (a small query CLI over the JSON) should print
sensible names, not raw ids. A newer build may change some values; the structure and the
derivations shall not.

## Lessons

See [SPEC-0039](../lessons/SPEC-0039-game-data-lessons.md). Top two for this part:

- **Refiner seconds are not `TimeToMake`.** Divide by 250 for substance outputs and 2 for product
  outputs (`RefinerSubsMadeInTime` / `RefinerProductsMadeInTime`); and check an ETA against two
  saves of one refiner before trusting it.
- **Stack size caps refiner output**: 9,999 for substances but 10 for products, so a gas -> product
  refiner stalls within minutes. The tables do not say this in one place; projections must.
