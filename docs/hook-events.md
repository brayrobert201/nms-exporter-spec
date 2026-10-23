# Hook events

Every file and line type the in-game mods write (live-hook area, SPEC-0040 to SPEC-0049). All files are
in the game prefix at `C:\pymhf\mod_saves\live\` (host: `<prefix>/drive_c/pymhf/mod_saves/live/`).

Common conventions:

- One JSON object per line in `*.jsonl` (UTF-8, `\n`-terminated), appended; `live.json` is a single
  object replaced atomically.
- `YYYYMMDD` and day boundaries are the local date of the game process (Wine inherits the host's time
  zone) at the moment of writing.
- `t` is Unix time in seconds as a float (1 decimal in `live.json` / trail / weather, 2 decimals in
  visor / systems), from the Windows Python's `time.time()`.
- Vectors are `[x, y, z]` floats. World coordinates are the game's; `rel` is world minus the nearest
  planet's centre = the save's planet frame (SPEC-0043).
- Enum fields carry the enum member name, or the raw integer when the value is not a member (e.g.
  `building: 63`). Readers must accept both.
- Fields may be added; readers ignore unknown fields.

| File | Writer | Rate |
|---|---|---|
| `live.json` | live_position | every 0.25 s while `cGcPlayerEnvironment::Update` runs |
| `trail-YYYYMMDD.jsonl` | live_position | on > 2 m move or change of planet / location / address |
| `weather-YYYYMMDD.jsonl` | live_position | each storm state change |
| `visor-YYYYMMDD.jsonl` | visor_log | events |
| `systems-YYYYMMDD.jsonl` | galaxy_log | events; one `system` per address per session |

---

## Position sample (`live.json` and each `trail-*.jsonl` line)

The current sample is also served by pyMHF's HTTP API at `GET http://127.0.0.1:5000/LivePosition/sample` (live_position 0.5; same object as `live.json`, `null` before the first sample).

| Field | Type | Meaning |
|---|---|---|
| `t` | float | sample time, 1 decimal |
| `location` | string | `cGcPlayerEnvironment.meLocation` (`EnvironmentLocation`: None_, Default, SpaceStation, PlanetOnFoot, PlanetInShip, PlanetInVehicle, Underwater, Cave, IndoorInBase, Freighter (= corvette interior), FreighterInternals (= capital freighter), AbandonedFreighter, InFleet, InSpaceObject, Nexus, Anomaly). Ship in flight/space reports `Default`. |
| `location_stable` | string | `meLocationStable`, same enum |
| `planet_index` | int | `miNearestPlanetIndex`; appears 0-based; not the address planet |
| `distance_from_planet` | float | `mfDistanceFromPlanet`: height above the planet's nominal radius (m, 1 decimal) |
| `sealevel` | float | `mfNearestPlanetSealevel`: water surface above nominal radius; 0 = no sea |
| `in_atmosphere` | bool | `mbInsidePlanetAtmosphere` |
| `pos` | vec3 | `mPlayerTM.pos`, world, 3 decimals |
| `planet_pos` | vec3 | `mNearestPlanetPos`, nearest planet centre, world |
| `rel` | vec3 | `pos - planet_pos` |
| `at` | vec3 | `mPlayerTM.at`; facing is `-at` on foot, `+at` in `PlanetInShip` / `Default` |
| `up` | vec3 | `mUp`; equals `rel/|rel|` on a planet |
| `ua` | object or null | galactic address (below); null when nmspy has no player state (mid-game attach) |

`ua` object:

| Field | Type | Meaning |
|---|---|---|
| `reality` | int | `RealityIndex` (galaxy, 0-255) |
| `voxel` | [int, int, int] | `[VoxelX, VoxelY, VoxelZ]`, signed (x, z: 12-bit; y: 8-bit) |
| `system` | int | `SolarSystemIndex` (12-bit) |
| `planet` | int | `PlanetIndex` (4-bit); 0 = in the system, at no planet |

Packed: `(x&0xFFF) | (z&0xFFF)<<12 | (y&0xFF)<<24 | reality<<32 | system<<40 | planet<<52`.

### `live.json` only: `weather`

| Field | Type | Meaning |
|---|---|---|
| `weather.storm` | object or null | last storm transition since the mod loaded: `{"state": eStormState name, "t": float}` |
| `weather.sun_angle` | float or null | `cGcSky.mfSolarSunAngle`, raw, 4 decimals, updated once per second |
| `weather.planet` | object or null | the nearest planet's data (below), refreshed every 5 s |
| `weather.planet_indices_seen` | [int] | every `cGcPlanet.miPlanetIndex` whose `UpdateWeather` ran this session |

`weather.planet`:

| Field | Type | Meaning |
|---|---|---|
| `planet_index` | int | `miPlanetIndex` |
| `weather` | string | `mPlanetData.Weather.WeatherType` (e.g. Humid, Toxic) |
| `storm_frequency` | string | `Weather.StormFrequency`: None_, Low, High, Always |
| `intensity` | string | `Weather.WeatherIntensity`: Default, Extreme |
| `hazard` | object | `Temperature`, `Toxicity`, `Radiation`, `LifeSupportDrain`, `SpookLevel`: 6 floats each, slots Ambient, Water, Cave, Storm, Night, DeepWater |
| `substances` | [string] | Common, Uncommon, Rare substance IDs (e.g. `"YELLOW2"`, `"TOXIC1"`) |
| `life`, `creature_life` | string | `Life`, `CreatureLife` |
| `race` | string | `InhabitingRace` |
| `buildings` | string | `BuildingLevel` |
| `biome`, `biome_subtype` | string | `GenerationData.Biome`, `.BiomeSubType` |
| `planet_class`, `size` | string | `GenerationData.Class`, `.Size` |
| `tile_colours` | [[r, g, b]] | 23 terrain colours, each channel clamped to 0..1, 3 decimals |

Example (trimmed):

```json
{"t": 1792701931.8, "location": "PlanetOnFoot", "location_stable": "PlanetOnFoot", "planet_index": 2,
 "distance_from_planet": 41.1, "sealevel": 0.0, "in_atmosphere": true,
 "pos": [150970.328, -4243.473, 832553.0], "planet_pos": [170000.0, -7000.0, 929000.0],
 "rel": [-19029.672, 2756.527, -96447.0], "at": [-0.952, -0.246, 0.181], "up": [-0.193, 0.028, -0.981],
 "ua": null,
 "weather": {"storm": null, "sun_angle": 186.2027, "planet_indices_seen": [0, 1, 2],
   "planet": {"planet_index": 2, "weather": "Toxic", "storm_frequency": "None_", "intensity": "Default",
     "hazard": {"Temperature": [11.0, 4.0, 4.0, 0.0, 5.0, 4.0], "...": "..."},
     "substances": ["YELLOW2", "TOXIC1", "ASTEROID1"], "life": "Full", "creature_life": "Mid",
     "race": "Warriors", "buildings": "Full", "biome": "Toxic", "biome_subtype": "Variant_A",
     "planet_class": "Initial", "size": "Medium", "tile_colours": [[0.192, 0.165, 0.125], "..."]}}}
```

Staleness: a reader treats `live.json` with `now - t > 10` s as stale (game stopped, paused, or mod
detour disabled).

---

## `weather-YYYYMMDD.jsonl`

One line per `cGcSky::SetStormState` call.

| Field | Type | Meaning |
|---|---|---|
| `state` | string | `eStormState`: Inactive, Warning, TransitionIn, Active, TransitionOut |
| `t` | float | time of the transition |
| `planet_index` | int or null | last `miNearestPlanetIndex` seen by the position hook |

Normal cycle: Warning -(25 s)-> TransitionIn -(20 s)-> Active -(length)-> TransitionOut -(20 s)->
Inactive -(gap)-> Warning. States can be skipped (Active straight from Inactive was seen).

---

## `visor-YYYYMMDD.jsonl`

Every line has `t` (2 decimals) and `event`. Most lines also carry `player` (the 1 s context):

| `player` field | Type | Meaning |
|---|---|---|
| `pos` | vec3 | `mPlayerTM.pos`, 2 decimals |
| `planet_pos` | vec3 | `mNearestPlanetPos` |
| `location` | string | `meLocation` |
| `at` | vec3 | `mPlayerTM.at` |

`player` is absent until the first context sample (first second after load).

### `visor_open`, `visor_closed`
No extra fields. Open on the first `cGcBinoculars::UpdateTarget`; closed 0.75 s after the last one.

### `scan_start`, `scan_complete`
No extra fields. From `cGcBinoculars::UpdateScanBarProgress` crossing 0 -> >0 and -> >=1.

### `scan_abandoned`
| Field | Type | Meaning |
|---|---|---|
| `reached` | float | progress (0..1, 2 decimals) when the bar fell back to 0 |

### `discovery_info`
From `cGcBinoculars::PopulateDiscoveryInfo`.
| Field | Type | Meaning |
|---|---|---|
| `submit` | bool | `lbSubmitDiscovery` |
| `silent` | bool | `lbSilent` |

### `visor_tag`
No extra fields. `cGcBinoculars::SetMarker` (player tagged a target).

### `marker_add`, `marker_remove`
From `cGcMarkerList::TryAddMarker` / `RemoveMarker`. Deduplicated per (event, type, world rounded to
1 m) for 60 s. Types NetworkPlayer, NetworkPlayerShip, NetworkPlayerVehicle, NetworkPlayerFireTeam,
NetworkPlayerFireTeamShip, NetworkPlayerFireTeamFreighter and SpacePoi are never logged.

| Field | Type | Meaning |
|---|---|---|
| `type` | string or int | `meType` (`cGcMarkerType`), e.g. TerrainResource (0x6), PlayerBase (0x22), Object, PowerHotspot (0x38), MineralHotspot (0x39), GasHotspot (0x3A) |
| `building` | string or int | `meBuildingClass` (`cGcBuildingClassification`); usually the raw `63` |
| `name` | string or null | `mCustomName` (e.g. `"Cobalt Deposit"`, `"UI_SURVEY_MINERALS_LABEL"`) |
| `subtitle` | string or null | `mCustomSubtitle` |
| `local` | vec3 | `mPosition.local` |
| `offset` | vec3 | `mPosition.offset`; world = local + offset |
| `rel` | vec3 | world - `player.planet_pos` (only when `player` exists) |
| `dist_from_player` | float | `|world - player.pos|`, 1 decimal (only when `player` exists) |
| `raw` | hex string | hotspot types only: the whole 0x2B0-byte marker struct, lowercase hex (1376 chars) |

A marker row has no address: assign it to the planet the trail says the player was on at `t`.
The hotspot class (C/B/A/S) is not in any field (see `label`).

### `marker_error`
Written once per session, on the first marker that could not be read.
| Field | Type | Meaning |
|---|---|---|
| `error` | string | `repr()` of the exception |

### `label` (chat `/hs ...`)
| Field | Type | Meaning |
|---|---|---|
| `rel` | vec3 or null | player position in the planet frame at the time; null if not yet known |
| `class` | "C" / "B" / "A" / "S" or null | first word of the command that is a class letter |
| `resource` | string or null | the other words, joined by single spaces, as typed (e.g. `"silver"`) |

Applies to the nearest hotspot within 50 m on the player's planet.

### `mark` (chat `/mark ...`)
| Field | Type | Meaning |
|---|---|---|
| `name` | string | text after `/mark`, or `"mark"` |
| `rel` | vec3 or null | as for `label` |

### `chat_error`
| Field | Type | Meaning |
|---|---|---|
| `error` | string | `repr()` of the exception in the chat hook |

---

## `systems-YYYYMMDD.jsonl`

Every line has `t` (2 decimals) and `event`. No `player` context.

### `system`
One per universe address per session, from `cGcGalaxyAttributesAtAddress::Classify` (only if
`mbValid`).

| Field | Type | Meaning |
|---|---|---|
| `ua` | int | 64-bit universe address of the system (planet bits 0) |
| `ua_hex` | string | `"0x%X"` of `ua` |
| `voxel` | [int, int, int] | signed `[x, y, z]` unpacked from `ua` |
| `system` | int | bits 40-51 |
| `planet` | int | bits 52-55 (0 here) |
| `reality` | int | bits 32-39 |
| `economy` | string | `TradingData.TradingClass` (`cGcTradingClass`, e.g. HighTech, Scientific) |
| `wealth` | string | `TradingData.WealthClass` (`cGcWealthClass`) |
| `conflict` | string | `ConflictData` (`cGcPlayerConflictData`: Default, Low, ...) |
| `race` | string | `Race` (`cGcAlienRace`: Warriors, Explorers, Traders, ...) |
| `star` | string | `Type` (`cGcGalaxyStarTypes`: Yellow, Blue, ...) |
| `anomaly` | string | `Anomaly` (`cGcGalaxyStarAnomaly`; None_ normally) |
| `planets` | int | `NumberOfPlanets` |
| `prime_planets` | int | `NumberOfPrimePlanets` |
| `space_pois` | int | `NumberOfSpacePois` |
| `planet_seeds` | [int] | `PlanetSeeds[i].Seed`, unsigned 64-bit, first `min(planets, 16)` |
| `abandoned`, `pirate`, `gas_giant`, `giant`, `safe` | bool | `AbandonedSystem`, `IsPirateSystem`, `IsGasGiantSystem`, `IsGiantSystem`, `IsSystemSafe` |

### `popup`
The galaxy map cursor's system changed (`cGcGalaxyMap::Data::DoSolarPopup`).
| Field | Type | Meaning |
|---|---|---|
| `ua` | int | `lQueryResult.mu64UA` |
| `ua_hex` | string | hex of `ua` |

### `system_name`
Once per (seed, name) per session (`cGcNameGenerator::GenerateSolarSystemName`).
| Field | Type | Meaning |
|---|---|---|
| `seed` | int | unsigned 64-bit; equals the system's universe address |
| `name` | string or null | generated name |
| `loc_name` | string or null | localised result (usually null) |
| `popup_ua` | int or null | UA of a `popup` less than 1 s earlier |

### `planet_name`
Once per (seed, name) per session (`cGcNameGenerator::GeneratePlanetName`).
| Field | Type | Meaning |
|---|---|---|
| `seed` | int | unsigned 64-bit; one of the system's `planet_seeds`; address planet n uses `planet_seeds[n-1]` |
| `name` | string or null | generated name |
| `loc_name` | string or null | localised result |

### `system_map_open`, `system_map_closed`, `system_map_marker`
No extra fields. Open on the first `cGcFrontendPageSolarSystemMap::Update`; closed 0.75 s after the
last; marker on `OnMarkerPlaced`.

### `error`
Once per hook per session.
| Field | Type | Meaning |
|---|---|---|
| `where` | string | `classify`, `popup`, `system_name`, `planet_name` |
| `error` | string | `repr()` of the exception |
