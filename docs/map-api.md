# Map API

Every endpoint the `/map` and `/systems` pages use. All are served by the exporter's own HTTP
server (default port 9877, configurable). JSON responses carry `Cache-Control: no-store`.
Query values are percent-encoded by the browser's `URLSearchParams`, which writes a space as `+`:
the server must decode them with `unquote_plus` semantics.

Conventions:

- **Address**: a universe address as hex with `0x` (`"0x1117EFC6D1114"`), the save's
  `GalacticAddress` packing (planet in bits 52–55, system 40–51, reality 32–39, Y 24–31,
  Z 12–23, X 0–11). Endpoints parse it base 16 (with or without `0x`) or as decimal.
- **w**: a planet-frame position `[x, y, z]` in metres from the planet centre (north = +Y).
- **x, y**: planet-view map metres (east, north) on the planet's azimuthal equidistant
  projection (SPEC-0060-REQ-003).
- **e, n, h**: base-view metres east, north and up from the base computer (SPEC-0060-REQ-006).
- **Target key**: `b:<base name>`, `o:<base name>:<object index>`, or `w:<x>,<y>,<z>`.

## Store-down behaviour (503)

The history-backed endpoints (`/api/elevation`, `/api/finds`, `/api/sectors`,
`/api/systems`) answer:

```
HTTP 503
{"error": "<why the history store is unavailable>", "db": false}
```

when the history store is not configured, or when any store error happens while serving the
request. Clients treat `db === false` as "store down": show the banner, keep save-backed content.
`/api/map` never answers 503 for the store; it degrades (below). Unexpected server errors answer
500 `text/plain` `error: <message>`.

---

## GET /map, GET /systems

HTML pages, read from disk on each request. `GET /` with `Accept: text/html` redirects 302 to
`/map`.

## GET /api/map

The save's planets with bases, plus base-less planets visited with the live hook.

```json
{
  "north_verified": true,
  "planets": [{
    "address": "0x…", "radius": 98304, "label": "Silver Ridge + 1 more",
    "bases": ["Silver Ridge", "Other"],
    "visited": true,                       // only on base-less planets from the history
    "planet_name": "Okyota Gamma" | null, "system_name": "Ubunji" | null,
    "portal": "117EFC6D1114" | null,       // SPEC-0068-REQ-006
    "tile_colours": [[r, g, b], …] | null, // 23 triples, 0–1
    "points": [
      {"kind": "base", "name": "…", "x": 0.0, "y": 0.0, "lat": 12.3, "lon": -45.6,
       "loose": 0, "power": "good|warning|critical|none" | null, "power_headline": "…" | null,
       "objects": 412, "w": [x, y, z]},
      {"kind": "exocraft", "name": "Nomad", "x": …, "y": …, "w": […]},
      {"kind": "deposit", "name": "Drained deposit", "x": …, "y": …, "w": […]},
      {"kind": "settlement", "name": "…", "x": …, "y": …, "w": […]}
    ],
    "layouts": {
      "<base name>": {
        "items": [{"kind": "extractor-mineral", "id": "U_EXTRACTOR_S", "e": 1.2, "n": -3.4,
                   "h": 0.5, "i": 17,
                   "poly": [[e, n], [e, n], [e, n], [e, n]], "top": 2.1,   // with footprints
                   "problems": ["…"], "fill": 120, "pipe_gap": 3.2,         // flagged only
                   "near": "4 m from a … network of …"}],
        "segments": [{"kind": "pipe|wire|link", "a": [e, n], "b": [e, n]}],
        "loose": 0,
        "footprints": true,
        "power": { …see SPEC-0066-REQ-006…, "status": "good", "headline": "…", "why": "…" } | null
      } | null
    }
  }],
  "errors": {"nms-db": "nms-db unreachable: …"}     // only when the history store is down
}
```

Planets are sorted by base count, most first. `power` fields: `extractors`, `demand_kp`,
`other_up_to_kp`, `em`, `em_kp` `[lo, hi]`, `solar`, `solar_kp`, `bio`, `bio_kp`, `batteries`,
`charge`, `capacity`, `charge_pct`, `battery_minutes`, `status`, `headline`, `why`.

**Degraded:** with the store down the response is still 200 with every planet that has bases;
base-less planets and names are missing and `errors["nms-db"]` explains.

## GET /api/live

Params: `base` (open base name, optional), `target` (target key, optional).

Always 200 when a live file exists, even with the history store down: the position is a file.
`db: false` plus `db_error` then say that the trail, names and finds are missing.

No live file:

```json
{"state": "none", "error": "no live.json in … (game not hooked?)"}
```

Off planet:

```json
{"state": "live|stale", "age": 1.2, "location": "Default", "error": null, "db": true,
 "weather": {…} | null, "address": "0x…" | null, "address_inferred": true,   // inferred only in attach mode
 "names": {"planet": "…" | null, "system": "…" | null},
 "on_planet": false, "system_bases": ["…"], "why": "12 km from the nearest planet · no bases in this system"}
```

On a planet:

```json
{"state": "live", "age": 0.3, "location": "PlanetOnFoot", "error": null, "weather": {…},
 "address": "0x…", "names": {…}, "on_planet": true, "has_bases": true,
 "player": {"x": 10.5, "y": -3.2, "heading": 87.0 | null, "lat": …, "lon": …, "w": [x, y, z]},
 "trail": [[x, y], …],                                  // last 6 h on this planet, ≤ 4000
 "in_base": {"name": "…", "e": …, "n": …, "h": …, "trail": [[e, n], …]},   // when base given
 "nearby": [{"key": "o:Base:17", "base": "Base", "kind": "depot", "id": "U_SILO_S",
             "dist": 11.0, "bearing": 140.2, "turn": -32.5 | null, "h": -2.1}],   // ≤ 12, ≤ 3 per kind
 "bases": [{…same shape, kind "base"…}],                // every base on the planet by distance
 "target": {"key": "…", "x": …, "y": …, "e": …, "n": …, "dist": …, "bearing": …, "turn": …, "h": …}
         | {"key": "…", "error": "not on this planet"}}
```

`state` is `stale` when the sample is older than 10 s. `weather` is the live hook's weather block
with resource IDs named, hazard tables keyed by slot, and `storm_eta` (live-hook area). Find
rows in `nearby` have `key: "w:x,y,z"`, `base: "class B · Silver"`, `kind` the find kind and
`id` the marker name.

**503** when the store is down (the trail and history live there).

## GET /api/elevation

Params: `address` (required), `cell` metres (default 25, clamped 2–500), `base` (optional: bin in
that base's frame).

```json
{"cell": 25, "radius": 98304, "water": 120 | null, "samples": 986, "objects": 2,
 "min": 37.0, "max": 265.0, "cells": [[i, j, h], …], "error": null}
```

Cell `(i, j)` covers `[i·cell, (i+1)·cell) × [j·cell, (j+1)·cell)` in map metres; `h` is the
lowest ground height in it, from the water surface when `water` is set, else from the nominal
radius. Errors (200): `{"error": "not known until you visit this planet with the live_position mod
running", "cells": []}`, `{"error": "no layout for <base>", "cells": []}`. **503** store down.

## GET /api/finds

Params: `address` (required), `base` (optional: add base-frame coordinates).

```json
{"finds": [{"kind": "hotspot-mineral|hotspot-gas|hotspot-power|deposit|mark",
            "name": "MineralHotspot label or deposit name or mark name",
            "w": [x, y, z], "key": "1117EFC6D1114:hotspot-mineral:123:-45:678",
            "first": 1761170844.0, "last": 1761171000.0,
            "class": "B" | null, "class_source": "you" | null, "has_raw": true,
            "rates": {"C": 250, "B": 375, "A": 500, "S": 625}, "rate_unit": "items/h per extractor",
            "resource": "Silver" | "Copper or Ammonia or Silver" | null,
            "options": ["Copper", "Ammonia", "Silver"],          // when ambiguous
            "basis": "recorded by you from the visor",
            "x": …, "y": …, "e": …, "n": …, "h": …}],             // e/n/h only with base
 "range_m": [190, 225],
 "ranges": {"hotspot-mineral": [190, 225], "hotspot-gas": [190, 225], "hotspot-power": [200, 250]},
 "unmatched_labels": [{"t": …, "class": "S", "resource": "silver"}]}
```

`rates` for power are kP per class strength; `rate_unit` is `"kP"`. Range and rate values come
from the game tables (game-data area); the numbers above are illustrative. A planet with no
projection returns `{"finds": []}`. **503** store down.

## POST /api/finds/class

Body (JSON, ≤ 4096 bytes): `{"key": "<find key>", "class": "C|B|A|S|null"}` and/or
`{"resource": "<name>|null"}`. `null` (or empty) clears the field.

- 200 `{"ok": true}`
- 400 `{"ok": false}` for an unknown key, a bad class, or no field; `bad json` (text) for an
  unparsable body
- 503 `{"ok": false, "error": "nms-db unreachable: …"}`

## GET /api/sectors

Params: `address` (required).

```json
{"radius": 98304, "depth": 6, "cells_per_sector": 4096, "cell_km": 1.21,
 "you": "N3W" | null,
 "total": {"charted": 8, "cells": 131072, "pct": 0.0061},
 "sectors": [{"id": "N3W", "name": "N3 west", "path": "21", "quadrant": "90–180°W",
              "corners": [[ux, uy, uz] ×3], "centre": [ux, uy, uz],
              "charted": 8, "pct": 0.195, "first": … | null, "last": … | null,
              "bases": ["…"], "finds": 38,
              "edge_xy": [[x, y] | null, …],      // 3 edges × 17 samples; null = far side
              "centre_xy": [x, y] | null}],       // 32 entries, N1P … S4C
 "cells": [{"sector": "N3W", "corners": [[ux, uy, uz] ×3], "first": …, "last": …,
            "xy": [[x, y] ×3]}],
 "error": null}
```

Corners are unit vectors in the planet frame. Error (200): `{"error": "not known until you visit
this planet with the live_position mod running", "sectors": [], "cells": []}`. **503** store down.

## GET /api/systems

Params: `from` (optional address of any point in the player's current system).

```json
{"ly_per_voxel": 400, "known_runes": 15 | null,
 "systems": [{"ua": "0x…", "name": "…" | null, "economy": "Scientific", "wealth": "Average",
              "conflict": "Low|Default|High|Pirate", "race": "Explorers", "star": "Blue",
              "anomaly": "None_" | "…", "planets": 2, "planet_names": ["…", null],
              "pirate": false, "abandoned": false, "here": true,
              "distance_ly": 0 | null, "voxel": [x, y, z], "bases": ["…"],
              "portal": "117EFC6D1114", "coords": "0913:007B:0ED0:017E",
              "dialable": true | null, "missing_glyphs": ["a"]}]}
```

Sorted by distance (unknown last), then name. `known_runes` is the save's `KnownPortalRunes`
bitmask (bit n = glyph n). **503** store down.

## GET /icon/<name>.png, GET /glyph/<hex>.png

Game HUD icons and the 16 portal glyphs, served from directories of PNGs the user extracts from
their own installed game (game-data area); they are not part of any repository. `<name>` is
`[A-Za-z0-9._]+` without `..`; `<hex>` is one of `0-9a-f`. Missing file: 404 `text/plain`
(`no icon …` / `no glyph …`). The map probes `HEAD /icon/base.png` at load and falls back to
shapes when it fails; glyph `<img>`s replace themselves with the hex digit on error.

Icon names used: `base`, `vehicle`, `settlement`, `pickup.terrain`, `pickup.mineral`,
`custommarker`, `surveying.minerals`, `surveying.gas`, `surveying.power`, each surveying icon
also with a class suffix `c|b|a|s` (`surveying.mineralsb`).
