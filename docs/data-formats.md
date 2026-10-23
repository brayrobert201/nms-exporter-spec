# Data formats

The formats shared between areas: what the map uses from the save, how a universe address is
packed, what the live hook writes (`live.json` and the day files), and how it all lands in
Postgres. Owners: the save side is SPEC 0001–0019, the hook SPEC 0040–0059, the map SPEC
0060–0079, the store SPEC 0080–0099 (normative DDL in SPEC-0080). Where this file and a spec
disagree, the spec wins; fix this file.

## Coordinate frame

Every position the map draws is in the **planet frame**: metres, origin at the planet centre,
axes as the game's world axes, **+Y is north** (`maps.NORTH_AXIS = (0, 1, 0)`). A base's saved
`Position` is already in this frame. The hook's `rel` is `player position - nearest planet
centre`, which is the same frame. For a point at unit vector `up` from the centre, north is +Y
projected onto the tangent plane and east is `north x up`.

Heights: the hook's `distance_from_planet` is the height above the planet's nominal radius, so
`|rel| - distance_from_planet` is the same number for every sample on a planet (seen: 130048 =
127 x 1024, 98304 = 96 x 1024). `sealevel` is the water surface above that radius (0 on dry
planets).

## Universe address (64-bit)

The save's `GalacticAddress` for bases, deposits and vehicles, the hook's `ua` dict once packed,
the galaxy log's `ua`, `event.address` and `planet_facts.address` are all the same 64-bit
integer:

| Bits | Field | Width | Notes |
|---|---|---|---|
| 0–11 | VoxelX | 12 | two's complement (-2048..2047) |
| 12–23 | VoxelZ | 12 | two's complement |
| 24–31 | VoxelY | 8 | two's complement (-128..127) |
| 32–39 | reality (galaxy index) | 8 | capped at 0xFF when packing |
| 40–51 | solar system index | 12 | capped at 0xFFF |
| 52–55 | planet index | 4 | 0 = the system itself / not on a planet; 1 and up = the planet |
| 56–63 | unused | 8 | 0 |

Packing the hook's dict `{"reality", "voxel": [x, y, z], "system", "planet"}`:

```
(x & 0xFFF) | (z & 0xFFF) << 12 | (y & 0xFF) << 24 | min(reality, 0xFF) << 32
  | min(system, 0xFFF) << 40 | planet << 52
```

Unpacking (sign-extend each voxel field):
`x = s12(a & 0xFFF)`, `z = s12((a >> 12) & 0xFFF)`, `y = s8((a >> 24) & 0xFF)`,
`reality = (a >> 32) & 0xFF`, `system = (a >> 40) & 0xFFF`, `planet = (a >> 52) & 0xF`.

Derived forms:

- **System address**: `a & ~(0xF << 52)` (planet bits cleared). Star systems are keyed by it.
- **Portal address** (12 hex digits, one per glyph): `planet (1, with 0 -> 1)`, `system (3)`,
  `Y (2)`, `Z (3)`, `X (3)`, each from the raw (unsigned) bit fields:
  `f"{p or 1:X}{(a>>40)&0xFFF:03X}{(a>>24)&0xFF:02X}{(a>>12)&0xFFF:03X}{a&0xFFF:03X}"`.
  The reality byte is not part of it.
- **Signal-booster coordinates** `XXXX:YYYY:ZZZZ:SSSS`: `x + 0x7FF`, `y + 0x7F`, `z + 0x7FF`,
  `system`, each 4 hex digits, from the signed voxel values.
- **Distance between systems**: one voxel is taken as 400 light years (`ly_per_voxel` in
  `/api/systems`); a convention, not from the game's data.

On the wire: the save may hold the address as an integer or as a `"0x..."` hex string; readers
accept both (`int(a, 16)` for strings). JSON APIs emit it as `"0x<HEX>"`. `planets.json` (legacy)
keyed it as a decimal string. The `find_label` key embeds it as bare uppercase hex.

## Save-side fields the map uses

After the save is decoded and its keys de-obfuscated (SPEC 0001–0019), from `PlayerStateData`:

| Field | Used for |
|---|---|
| `UniverseAddress.RealityIndex`, `UniverseAddress.GalacticAddress.{VoxelX, VoxelY, VoxelZ, SolarSystemIndex, PlanetIndex}` | where the save says the player is |
| `PersistentPlayerBases[]` | one entry per base, kept as `raw_bases` |
| `  .Name` | label; `?base=` lookups |
| `  .BaseType.PersistentBaseTypes` | skip `FreighterBase`, `PlayerShipBase`, `PlayerSpaceBase` (not on a planet) |
| `  .GalacticAddress` | which planet (universe address) |
| `  .Position` | base origin in the planet frame |
| `  .Forward` | base orientation (made orthogonal to `Position` to get the base's axes) |
| `  .LastUpdateTimestamp` | freshness |
| `  .Objects[].{ObjectID, Position, Up, At, UserData}` | layout: parts in base-local coordinates; extractor/depot fill in the upper 32 bits of `UserData` |
| `VehicleOwnership[i].{Location, Position}` | exocraft; `i` is the type: Roamer, Nomad, Colossus, Pilgrim, (unused), Nautilon, Minotaur |
| `DrainedTerrainResourceDeposits[].{GalacticAddress, Position}` | drained deposits |
| `SettlementStatesV2[].{UniverseAddress, Position, Name}` | settlements |
| `KnownPortalRunes` | bitmask, one bit per glyph 0–F |

A point is drawn on a planet only if its address matches and `| |Position| - radius | < 3 %` of
the radius (vehicles parked in space are skipped).

## The hook's output directory

Full line-type reference, with writers and rates: `docs/hook-events.md` (SPEC 0040–0059). The
subset below is what the store and the map depend on.

The hook writes into one directory inside the game's Proton prefix
(`<prefix>/drive_c/pymhf/mod_saves/live`; the exporter's `NMS_LIVE_DIR`). Day files are named by
the local date when the line was written and only ever appended to.

### `live.json` (state, not ingested)

Replaced atomically (write `live.json.tmp`, rename) every 0.25 s while the game runs. Readers treat
it as stale after 10 s.

```json
{
  "t": 1761200000.1,
  "location": "PlanetOnFoot",          // the game's location enum name
  "location_stable": "PlanetOnFoot",
  "planet_index": 1,
  "distance_from_planet": 152.3,       // m above nominal radius
  "sealevel": 0.0,
  "in_atmosphere": true,
  "pos": [x, y, z],                    // world
  "planet_pos": [x, y, z],             // nearest planet centre, world
  "rel": [x, y, z],                    // pos - planet_pos: the planet frame
  "at": [x, y, z],                     // facing is -at on foot, +at in a ship
  "up": [x, y, z],
  "ua": {"reality": 0, "voxel": [x, y, z], "system": 123, "planet": 1},   // or null
  "weather": {
    "storm": {"state": "...", "t": 1761199000.0},   // or null
    "sun_angle": 0.1234,
    "planet": {                         // nearest planet, refreshed every 5 s; or null
      "planet_index": 1, "weather": "...", "storm_frequency": "...", "intensity": "...",
      "hazard": {"Temperature": [6 floats], "Toxicity": [...], "Radiation": [...],
                 "LifeSupportDrain": [...], "SpookLevel": [...]},
      "substances": ["<common id>", "<uncommon id>", "<rare id>"],
      "life": "...", "creature_life": "...", "race": "...", "buildings": "...",
      "biome": "...", "biome_subtype": "...", "planet_class": "...", "size": "...",
      "tile_colours": [[r, g, b] x 23]           // 0..1
    },
    "planet_indices_seen": [1, 2]
  }
}
```

`ua` is null when the hook was attached mid-game and could not find the galactic address; the
store then infers the planet from `planet_pos` (SPEC-0080 REQ-012).

### `trail-YYYYMMDD.jsonl`

The same object as `live.json` **without** the `weather` block, appended whenever the player moved
more than 2 m in the planet frame, or `planet_index`, `location` or `ua` changed.

### `visor-YYYYMMDD.jsonl`

Every line: `{"t": <unix s, 2 dp>, "event": "<kind>", ...}`. Kinds and their extra fields:

| `event` | Extra fields |
|---|---|
| `marker_add`, `marker_remove` | `type` (marker type name, e.g. `MineralHotspot`, `GasHotspot`, `PowerHotspot`, `TerrainResource`, `PlayerBase`), `building`, `name`, `subtitle`, `local`, `offset` (raw position parts), `rel` (local + offset - planet centre; when the player position is known), `dist_from_player`, `raw` (hex of the struct, some types only) |
| `label` | `rel`, `class` (C/B/A/S or null), `resource` (or null): the `/hs` chat command |
| `mark` | `name`, `rel`: the `/mark` chat command |
| `visor_open`, `visor_closed`, `scan_start`, `scan_complete`, `scan_abandoned` (`reached`), `discovery_info` (`submit`, `silent`), `visor_tag` | as named |
| `marker_error`, `chat_error` | `error` |

The hook itself suppresses a marker it logged within its repeat window; the store additionally
drops any `marker_add` whose `(type, name, round(rel/5 m))` it has already stored.

### `systems-YYYYMMDD.jsonl`

Every line: `{"t", "event", ...}`.

| `event` | Extra fields |
|---|---|
| `system` | `ua` (int), `ua_hex`, `voxel` [x, y, z] (signed), `system`, `planet`, `reality`, `economy`, `wealth`, `conflict`, `race`, `star`, `anomaly`, `planets`, `prime_planets`, `space_pois`, `planet_seeds` (uint64 list, length `planets`, max 16), `abandoned`, `pirate`, `gas_giant`, `giant`, `safe` |
| `popup` | `ua`, `ua_hex`: the system under the galaxy-map cursor |
| `system_name` | `seed` (uint64), `name`, `loc_name`, `popup_ua`: a system name's seed is its system address |
| `planet_name` | `seed`, `name`, `loc_name`: a planet name's seed is one of its system's `planet_seeds` (planet index n names `planet_seeds[n-1]`) |
| `system_map_open`, `system_map_closed`, `system_map_marker`, `error` | as named |

### `weather-YYYYMMDD.jsonl`

`{"state": "<storm state name>", "t": <unix s>, "planet_index": <int>}`, one line per storm
transition.

### Legacy files (imported once, then unused)

- `planets.json`: `{"<decimal address>": {"substances", "biome", "weather", "storm_frequency",
  "intensity", "tile_colours"}}`.
- `find_labels.json`: `{"<find key>": {"class"?, "resource"?, "t"}}`.

## Postgres

Normative DDL: SPEC-0080. Summary:

| Object | Columns | Notes |
|---|---|---|
| `event` | `id bigserial`, `stream text`, `t float8`, `address bigint NULL`, `data jsonb` | one row per ingested line; `data` is the line unchanged; `address` set for trail rows only |
| `ingest_file` | `path text PK`, `pos bigint`, `updated timestamptz` | basename -> byte offset after the last ingested newline |
| `find_label` | `key text PK`, `label jsonb` | key `"<ADDRESS HEX>:<kind>:<x5>:<y5>:<z5>"` where kind is `hotspot-mineral`, `hotspot-gas`, `hotspot-power`, `deposit` or `mark` and `x5` = `round(x / 5)`; label `{"class"?: "C"/"B"/"A"/"S", "resource"?: str, "t": unix s}` |
| `planet_facts` | `address bigint PK`, `facts jsonb`, `updated` | facts = the `substances`, `biome`, `weather`, `storm_frequency`, `intensity`, `tile_colours` of `live.json`'s `weather.planet` |
| `schema_version` | `version int` | 1 |
| view `trail` | `id, at, t, address, planet, location, x, y, z, altitude, sealevel` | `planet = (address >> 52) & 15` |
| view `marker` | `id, at, t, type, name, x, y, z` | `marker_add` only |
| view `chat` | `id, at, t, command, class, resource, name, x, y, z` | `label` and `mark` |
| view `star_system` | `id, at, data` | `systems` stream, `event = 'system'` |
| view `storm` | `id, at, data` | `weather` stream |

Locks: `pg_advisory_xact_lock(4207)` around schema changes, `pg_try_advisory_lock(4208)` (session)
for the single ingest writer.

Example queries:

```sql
-- where was I yesterday evening
SELECT at, planet, location, round(x) x, round(y) y, round(z) z FROM trail
WHERE at BETWEEN now() - interval '1 day' AND now() - interval '20 hours' ORDER BY id;
-- hotspots seen, newest first
SELECT at, type, name FROM marker WHERE type LIKE '%Hotspot' ORDER BY id DESC LIMIT 20;
-- systems seen per star colour
SELECT data->>'star', count(*) FROM star_system GROUP BY 1;
```

## Error body when the store is down

`{"error": "<reason>", "db": false}` with HTTP 503 on the history routes; `{"ok": false, "error":
"..."}` on `POST /api/finds/class`; `errors["nms-db"]` inside a 200 `/api/map`. SPEC-0081.
