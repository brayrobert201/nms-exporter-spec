# Game data locations

Where in No Man's Sky's `.pak` archives the things this project reads live, and what each holds.
Observed on Steam build 25800958 (Cosmos 7.06), MBINCompiler 7.04.1.3. Paths are the packed paths
as `hgpaktool -L` lists them (lower case, forward slashes); MBINCompiler converts each `.mbin` to an
`.MXML` beside it. A game update can move or rename any of these: re-find with the pak index
(SPEC-0020) rather than trusting this table.

The archives are in `<game dir>/GAMEDATA/PCBANKS/*.pak` (97 `.pak` files plus `BankSignatures.bin`; about 195,000 packed paths
in total). The Steam build id is the `"buildid"` value in `steamapps/appmanifest_275850.acf`
(275850 is the game's Steam app id).

Nothing extracted from these files may be redistributed: models, textures, icons, fonts and text are
Hello Games' property. Specs here describe how to extract; the rebuild extracts on the user's own
machine from their own installed copy.

## Tables and globals (SPEC-0021, SPEC-0022)

| Packed path | Pak | Holds | Fields read |
|---|---|---|---|
| `metadata/reality/tables/nms_reality_gcrecipetable.mbin` | Precache | Refiner and cooking recipes | `Table` > entries: `Id`, `RecipeType`, `RecipeName`, `TimeToMake`, `Cooking`, `Result{Id,Type.InventoryType,Amount}`, `Ingredients[]` |
| `metadata/reality/tables/nms_reality_gcsubstancetable.mbin` | Precache | Raw substances (`GcRealitySubstanceData`) | `ID`, `Name`/`NameLower`, `Symbol`, `Category`, `BaseValue` |
| `metadata/reality/tables/nms_reality_gcproducttable.mbin` | Precache | Products (`GcProductData`) | `ID`, `Name`/`NameLower`, `Subtitle`, `Type`, `BaseValue`, `IsCraftable`, `Requirements[]`, `CraftAmountMultiplier`, `Cost{LowPriceMod,HighPriceMod,SpaceStationMarkup}`, `EconomyInfluenceMultiplier` |
| `metadata/reality/tables/nms_basepartproducts.mbin`, `nms_modularcustomisationproducts.mbin`, `proceduralproducttable.mbin` | Precache | Base parts, corvette/modular parts, procedural products, all as `GcProductData` | as products |
| `metadata/reality/tables/nms_reality_gctechnologytable.mbin` | Precache | Technologies (`GcTechnology`) | `ID`, `Name`, `BaseStat`, `Category`, `Core`, `Upgrade` |
| `metadata/reality/tables/nms_reality_gcproceduraltechnologytable.mbin` | Precache | Upgrade modules (`GcProceduralTechnologyData`) | `ID`, `Template`, `Quality`, `Name`/`NameLower` |
| `language/nms_*_english.mbin` (`nms_loc1_english` ... `nms_loc10_english`, `nms_update3_english`, ...) | MetadataEtc | Localised strings (`TkLocalisationEntry`) | `Id`, `English` |
| `gcgameplayglobals.global.mbin` | globals | Refiner speed divisors, survey range | `RefinerSubsMadeInTime`, `RefinerProductsMadeInTime`, `...Survival`, `SurveyMaxDistance` |
| `gcenvironmentglobals.global.mbin` | globals | Hotspot falloff | `MaxHotspotOffsetDistance`, `MinHotspotFalloffDistance`, `MaxHotspotFalloffDistance` |
| `metadata/simulation/scanning/regionhotspotstable.mbin` | Precache | Hotspot categories, classes, strengths, ranges, biome gases, extractable substances | `RegionHotspots`, `RegionHotspotBiomeGases`, `RegionHotspotSubstances` |
| `metadata/reality/tables/basebuildingobjectstable.mbin` | Precache | Every placeable base object (`GcBaseBuildingEntry`, 2,124 entries) | `ID`, `LinkGridData`, `PlacementScene.Filename` |
| `metadata/reality/tables/basebuildingpartstable.mbin` | Precache | Base parts and their models (`GcBaseBuildingPart`, ids prefixed `_`) | `Model.Filename` |
| `gcfleetglobals.global.mbin` | globals | Frigate expedition timing, trait strengths | `ExpeditionDurations`, `TimeTakenForExpeditionEvent`, `FrigateTraitStrengths` |
| `metadata/reality/tables/frigatetraittable.mbin` | Precache | Frigate traits (`GcFrigateTraitData`) | `ID`, `DisplayName`, `FrigateStatType`, `Strength` |
| `metadata/simulation/solarsystem/biomes/biomelistperstartype.mbin` | Precache | Biome weights per star colour | `StarType` > `BiomeProbability` |
| `metadata/reality/tables/rewardtable.mbin` | Precache | Reward tables, including crop harvests (`PLANT_*`) | `GcGenericRewardTableEntry`, `ID`, `AmountMin`, `AmountMax` |
| `metadata/simulation/missions/tables/*.mbin` (about 30 tables) | Precache | Mission sequences (`GcGenericMissionSequence`) | `MissionID`, `MissionTitles.Format`, `MissionClass`, `Stages`, `FinalStageVersions`, `Rewards`, `IsRecurring` |
| `gcskyglobals.globals.mbin` | globals | Storm timing | `MinStormLength{Low,High}`, `MaxStormLength{Low,High}`, `MinTimeBetweenStorms{Low,High}`, `MaxTimeBetweenStorms{Low,High}`, `StormWarningTime`, `StormTransitionTime`, extreme fallback |
| `metadata/simulation/solarsystem/weather/<kind>weather.mbin` (e.g. `humidweather`) | Precache | Per-weather hazard tables, Normal and Extreme | `cGcPlanetHazardData` |

"Pak" is the short name: MetadataEtc = `NMSARC.MetadataEtc.pak`, globals = `NMSARC.globals.pak`,
Precache = `NMSARC.Precache.pak`, TexUI = `NMSARC.TexUI.pak`. The table extractor (SPEC-0021)
unpacks from MetadataEtc, globals, misc and Precache with `hgpaktool -f` globs (listed in that spec).

## Art (SPEC-0023) — extract locally, never commit

| Packed path | Pak | Holds |
|---|---|---|
| `textures/ui/hud/icons/scanning/surveying.{minerals,gas,power}.dds` | `NMSARC.TexUI.pak` | Survey hotspot icon, class unknown |
| `textures/ui/hud/icons/scanning/surveying.{minerals,gas,power}{a,b,c,s}.dds` | `NMSARC.TexUI.pak` | Survey hotspot icon with class letter |
| `textures/ui/hud/icons/scanning/{pickup.mineral,building.portal,pickup.terrain}.dds` | `NMSARC.TexUI.pak` | Deposit, portal, terrain pickup markers |
| `textures/ui/hud/icons/player/{base,vehicle,settlement,custommarker,ship,corvette,freighter,playerlocation}.dds` | `NMSARC.TexUI.pak` | Place markers |
| `textures/ui/frontend/icons/update3/portalsymbol.{0..f}.dds` | `NMSARC.TexUI.pak` | The 16 portal glyphs |

## Scenes (SPEC-0024) — read for numbers only

| Packed path | Holds |
|---|---|
| `models/planets/biomes/common/buildings/parts/buildableparts/**/<name>_placement.scene.mbin` | Placement ("ghost") scene per base object: collision shapes, mesh AABBs, references |
| e.g. `.../buildableparts/utilityparts/module_generators_placement.scene.mbin` | Electromagnetic generator placement scene: one `Box` collision 2.757 x 7.166 x 2.757 |
| e.g. `.../buildableparts/basicparts/basic_wall_placement.scene.mbin` | Basic wall placement scene |
| e.g. `.../buildableparts/cuberoom_placement.scene.mbin` | Cuboid room placement scene |
| `.../buildableparts/utilityparts/powerline_connector.scene.mbin` | Snap-point scene referenced by many placement scenes (not geometry) |

Scene-internal references (`SCENEGRAPH` attribute values) are written upper case with backslashes,
e.g. `MODELS\PLANETS\...\POWERLINE_CONNECTOR.SCENE.MBIN`; normalise to lower case and `/` before
looking them up in the index.

## Entities (SPEC-0025)

| Packed path | Pak | Holds |
|---|---|---|
| `models/planets/biomes/common/buildings/parts/buildableparts/freighterbase/rooms/plantroom/parts/floor0/entities/interaction.entity.mbin` | Precache | Freighter Hydroponic Room floor: `UI_HARVEST_ALL` interaction, `GcHarvestPlantAction.Radius` 6.0 |
| `models/planets/biomes/common/buildings/parts/buildableparts/bioroom/entities/interaction.entity.mbin` | Precache | Biodome: same action, `Radius` 6.0 |
| `.../buildableparts/tech/cuberoom_plantermega.scene.mbin` | — | Large planter; a community mod adds Harvest All here (community claim) |

## Outside the paks

| Path | Holds |
|---|---|
| `<game dir>/GAMEDATA/MODS/<ModName>/<PACKED PATH UPPER CASE>` | Loose-file mods: an MBIN placed here replaces the packed file of the same path at start-up |
| `steamapps/appmanifest_275850.acf` | `"buildid" "NNNNNNNN"`: the cache and staleness key for everything above |
| MBINCompiler release asset `mapping.json` | Save-file key de-obfuscation map (used by the save reader, area save-exporter) |
