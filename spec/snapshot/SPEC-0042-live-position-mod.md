---
id: SPEC-0042
title: 'live_position mod: player position at 4 Hz, trail, galactic address'
description: 'pyMHF mod that hooks cGcPlayerEnvironment::Update and writes live.json every 0.25 s and a daily trail of moves over 2 m, plus the weather block (SPEC-0044).'
type: snapshot
area: live-hook
author: bob
created: 2026-10-23
last_updated: '2026-10-24'
status: current
depends_on: [SPEC-0040]
supersedes: []
superseded_by: []
related_specs: [SPEC-0043, SPEC-0044, SPEC-0048, SPEC-0049]
screenshots: []
---

# SPEC-0042: live_position mod: player position at 4 Hz, trail, galactic address

> Runs inside the game process (SPEC-0040). In launch mode it is loaded before the game boots; in attach mode, save before attaching.

## Summary

`live_position.py` is the mod the live map depends on. On every frame the game calls
`cGcPlayerEnvironment::Update`; the mod's after-detour samples it at most four times a second and
writes the player's position, the nearest planet's centre, facing and up vectors, location kind,
height above the planet and (when known) the galactic address to `live.json`, replaced atomically.
Each sample that moved more than 2 m, or changed planet, location or address, is also appended to a
daily `trail-YYYYMMDD.jsonl`. The weather hooks in the same mod add a `weather` block (SPEC-0044).
What the numbers mean, and the facts the map derives from them, are in SPEC-0043.

## Inputs and outputs

- Hook: `cGcPlayerEnvironment.Update.after(this, lfTimeStep)` (nmspy names; `this` is a pointer to
  `cGcPlayerEnvironment`).
- Galactic address: `nmspy.common.gameData.player_state.mLocation` (nmspy fills `player_state` only
  after a game state change, so a mid-game attach often has none: `ua` is `null`).
- Output dir: `C:\pymhf\mod_saves\live\`:
  - `live.json` - latest sample plus `weather` block, rewritten every 0.25 s.
  - `trail-YYYYMMDD.jsonl` - one sample per line on movement.
  Field tables: `docs/hook-events.md`.

## Requirements

### SPEC-0042-REQ-001: Sampling
The mod shall sample in `cGcPlayerEnvironment.Update.after` at most once per `INTERVAL = 0.25` s of
wall time (`time.time()`), skipping calls with a null `this`.

### SPEC-0042-REQ-002: Fields read
Each sample shall read only these inline fields of `this.contents`:

| JSON field | Game field | Notes |
|---|---|---|
| `pos` | `mPlayerTM.pos` | world position, 3 decimals |
| `at` | `mPlayerTM.at` | unit vector; sign depends on location (SPEC-0043) |
| `up` | `mUp` | local up; equals normalised `rel` on a planet |
| `planet_pos` | `mNearestPlanetPos` | nearest planet's centre, world |
| `planet_index` | `miNearestPlanetIndex` | 0-based index in the system (see SPEC-0043) |
| `distance_from_planet` | `mfDistanceFromPlanet` | height above nominal radius, 1 decimal |
| `sealevel` | `mfNearestPlanetSealevel` | water surface above nominal radius |
| `in_atmosphere` | `mbInsidePlanetAtmosphere` | bool |
| `location`, `location_stable` | `meLocation`, `meLocationStable` | `EnvironmentLocation` enum name via `str()` |

and compute `rel = pos - planet_pos` (3 decimals) and `t = round(time.time(), 1)`.

### SPEC-0042-REQ-003: Galactic address
Unless a previous lookup failed, each sample shall set `ua` from `gameData.player_state`:
`{"reality": mLocation.RealityIndex, "voxel": [GalacticAddress.VoxelX, VoxelY, VoxelZ],
"system": SolarSystemIndex, "planet": PlanetIndex}`; `null` when `player_state` is `None`. If the
lookup raises once, the mod shall log it and stop trying for the rest of the session (`ua` stays
`null`). The mod shall not obtain the application object any other way (SPEC-0041 REQ-007): the
exporter infers a missing address from the planet centre (SPEC-0043 REQ-008).

### SPEC-0042-REQ-004: live.json
Each sample shall be written as one JSON object `{...sample, "weather": {...}}` to `live.json.tmp`
then `os.replace`d onto `live.json`, so a reader never sees a partial file.

### SPEC-0042-REQ-004a: Current sample over pyMHF's HTTP API (0.5)
The mod shall also expose the same object (`{...sample, "weather": {...}}`, or `null` before the first
sample) as a read-only property decorated with pyMHF's `@endpoint("sample", method="GET")`
(`from pymhf.core.http_api import endpoint`). pyMHF serves it at
`GET http://127.0.0.1:5000/LivePosition/sample`: the route is prefixed with the mod's class name, and
pyMHF only starts its uvicorn server (bound to `0.0.0.0:5000`) when some loaded mod declares an
endpoint. The import shall be guarded: without the `http_api` extra the decorator is a no-op and the
file remains the only output. `live.json` keeps being written as a fallback.

The suggestion came from pyMHF's author: state is a good fit for the endpoint; history (trail, visor,
systems, weather) stays as append-only files (SPEC-0048), because a pull API only has what the mod
holds at the moment it is asked.

### SPEC-0042-REQ-005: Trail
A sample shall be appended (one JSON line, same fields as `live.json` minus `weather`) to
`trail-YYYYMMDD.jsonl` (local date of the sample) when it is the first of the session, or
`(planet_index, location, ua)` differs from the last trail line's, or the squared distance between
the two `rel` vectors exceeds `MIN_MOVE^2` with `MIN_MOVE = 2.0` m.

### SPEC-0042-REQ-006: Exporter source and staleness
The exporter shall read the current sample from the HTTP endpoint first (timeout 0.3 s; any error,
non-object or missing `t` counts as no answer; `NMS_LIVE_URL=""` disables it) and fall back to
`live.json`. It shall treat the sample as `live` when `now - t <= STALE_S = 10` s and `stale` otherwise,
and report `none` with a reason when the file is missing or unparsable. Reads return
`(value, error)` and never raise.

### SPEC-0042-REQ-007: No side effects on the game
The mod shall never write game memory, never follow a pointer, and never raise out of a detour other
than the single failing-address case of REQ-003, which is caught.

### SPEC-0042-REQ-008: Mod metadata
The mod class shall be a `pymhf.Mod` decorated `@no_gui` (no DearPyGui panel), `__version__ = "0.4"`
at the time of writing; it creates the output dir in `__init__`.

## Mod shape

The hook signature and decorator form are the spec (nmspy 181442.1 / pyMHF 0.2.5):

```python
from pymhf import Mod
from pymhf.gui.decorators import no_gui
import nmspy.data.types as nms
from nmspy.common import gameData

@no_gui
class LivePosition(Mod):
    __version__ = "0.4"

    @nms.cGcPlayerEnvironment.Update.after
    def on_env_update(self, this, lfTimeStep):     # this: pointer; this.contents is the struct
        ...

    @nms.cGcSky.SetStormState.after
    def on_storm(self, this, leNewState): ...      # SPEC-0044

    @nms.cGcSky.Update.after
    def on_sky(self, this, lfTimeStep): ...        # SPEC-0044

    @nms.cGcPlanet.UpdateWeather.after
    def on_planet_weather(self, this, lfTimeStep): ...   # SPEC-0044
```

Constants: `OUT_DIR = r"C:\pymhf\mod_saves\live"`, `INTERVAL = 0.25`, `MIN_MOVE = 2.0`,
`PLANET_INTERVAL = 5.0`. Vectors are rounded to 3 decimals, `distance_from_planet` and `sealevel` to 1.
The mod deliberately has **no** `cGcApplication.Update` hook (SPEC-0041 REQ-007).

## Reader contract

- Readers take the current sample from `GET :5000/LivePosition/sample` when it answers, else poll
  `live.json` (the map polls about once a second); either way they must tolerate a missing sample,
  partial history, and `ua: null`.
- Readers ingest trail lines incrementally (remember a byte offset or line count per day file) and
  never rewrite the files; the mod may be appending at the same moment, so a final line without `\n`
  is left for the next read.
- The trail is the source for planet history: the elevation grid, nominal radius, the planet-centre ->
  address map (SPEC-0043) and the timeline that assigns visor markers to planets (SPEC-0045).
- Observed volume: one evening of play wrote 1.6 MB of trail (3,686 lines).

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| Game not running / not injected | No writes; `live.json` keeps its last content; exporter says `stale` after 10 s, `none` if the file never existed. |
| Game paused / in a menu that stops `cGcPlayerEnvironment::Update` | Same as above: stale. A stale feed is not proof of a crash; check the game. |
| Attached mid-game (attach mode only; launch mode has the address from the first sample) | `ua` null until nmspy sees a game state change (warp, load); the exporter infers the planet from `planet_pos` if seen before with an address, else shows "waiting for the game to report where you are". |
| Address lookup raised | `ua` null for the session; logged once. |
| Detour disabled by pyMHF | `live.json` stops; census and re-enable per SPEC-0041 REQ-005. |
| Disk full / write error | The exception would disable the detour; the feed goes stale. |
| In space / galaxy map | Samples continue; `planet_index` / `planet_pos` are the nearest planet's; the map refuses to place the player off-planet (SPEC-0043 REQ-007). |

## Verification

1. Attached, standing still: `live.json` `t` advances by ~0.25 s; the trail gets no new lines.
2. Walk 10 m: about five trail lines appear (one per > 2 m step).
3. Stand in a planetary base: `|rel|` matches the save's `PersistentPlayerBases[].Position` magnitude
   for that base to within tens of metres (measured: 21.6 m between player and base Position).
4. `up` equals `rel / |rel|` to 3 decimals (measured: (-0.792, -0.021, -0.610) both).
5. Walk north on the in-game compass: `rel.y` rises. Face east: the exporter's heading is ~90
   (measured 89.4). See SPEC-0043.
6. Kill the game: within 10 s the exporter's `/api/live` state becomes `stale`.

## Lessons

See [SPEC-0049](../lessons/SPEC-0049-live-hook-lessons.md): the app-object grab that crashed the game
(why `ua` can be null), and `at` reversing in ships.
