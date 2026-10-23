---
id: SPEC-0023
title: 'HUD icons and portal glyphs: extracting game art locally'
description: 'Two small extraction scripts that turn the game''s DDS textures for survey hotspots, place markers and the 16 portal glyphs into 64x64 PNGs on the user''s own machine, served by the exporter and never committed.'
type: snapshot
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0020]
supersedes: []
superseded_by: []
related_specs: [SPEC-0032, SPEC-0039, SPEC-0060, SPEC-0063, SPEC-0068, SPEC-0046]
screenshots: [planet-map-icons.png, portal-glyphs.png, hotspot-class-panel.png]
---

# SPEC-0023: HUD icons and portal glyphs

## Summary

The map (SPEC-0060–0069) reads better with the game's own symbols: a class-S gas hotspot drawn with
the HUD's own "gas S" icon is recognisable at a glance; a portal address drawn in the game's
glyphs can be dialled by eye. These images are Hello Games' art. They are therefore **extracted
on the user's machine from their installed copy** by two scripts, written to gitignored
directories, and served by the exporter from there; the repository holds only the extraction
recipe (SPEC-0032). Every page that uses them shall work without them.

## Inputs and outputs

**Inputs:** `NMSARC.TexUI.pak` via `nms-pak get` (SPEC-0020); an image converter that reads DDS
(the original uses ImageMagick's `convert`; any DDS-capable converter will do, e.g. Pillow is
*not* standard library, so the original shells out instead).

**Outputs:**

| Script | Writes | Served at |
|---|---|---|
| `nms-icons` | `data/icons/<name>.png`, 64 x 64 | `GET /icon/<name>.png` |
| `nms-glyphs` | `data/glyphs/<0-f>.png`, 64 x 64 | `GET /glyph/<hex digit>.png` |

Both directories are in `.gitignore`. The exporter's directories can be overridden with
`NMS_ICON_DIR` and `NMS_GLYPH_DIR`.

## Requirements

### SPEC-0023-REQ-001: Icon set

The icon script shall extract these textures from `textures/ui/hud/icons/`:

- `scanning/surveying.minerals`, `scanning/surveying.gas`, `scanning/surveying.power` (class
  unknown);
- `scanning/surveying.<t><c>` for `t` in `minerals, gas, power` and `c` in `a, b, c, s` (12
  icons; the class letter is part of the art);
- `scanning/pickup.mineral` (resource deposit), `scanning/building.portal`,
  `scanning/pickup.terrain`;
- `player/base`, `player/vehicle`, `player/settlement`, `player/custommarker`, `player/ship`,
  `player/corvette`, `player/freighter`, `player/playerlocation`.

That is 26 icons. The output file name is the texture's basename without `.dds`
(e.g. `surveying.gasa.png`, `base.png`).

### SPEC-0023-REQ-002: One unpack call

The icon script shall build a single alternation regex of all names with `.` escaped, anchored as
`textures/ui/hud/icons/(<n1>|<n2>|...)\.dds$`, and call `nms-pak get <regex> --all` once; then
for each name pick the first output line ending in `/<name>.dds`.

### SPEC-0023-REQ-003: Glyph set and order

The glyph script shall extract
`textures/ui/frontend/icons/update3/portalsymbol.<h>.dds` for `h` in `0123456789abcdef` (regex
`textures/ui/frontend/icons/update3/portalsymbol\.[0-9a-f]\.dds$`) and write `<h>.png`. The glyph
for hex digit `h` is the one the game uses for that value in a portal address. Names in order
0–F: **Sunset, Bird, Face, Diplo, Eclipse, Balloon, Boat, Bug, Dragonfly, Galaxy, Voxel, Fish,
Tent, Rocket, Tree, Atlas.**

The portal address itself (12 hex digits: planet 1, system 3, Y 2, Z 3, X 3, i.e.
`"%X%03X%02X%03X%03X" % (planet & 0xF, system & 0xFFF, y & 0xFF, z & 0xFFF, x & 0xFFF)`) and the
save's `KnownPortalRunes` bitmask (bit n = glyph n learned) are specified with the systems page
(SPEC-0068) and the galaxy log (SPEC-0046).

### SPEC-0023-REQ-004: Conversion

Each texture shall be converted from DDS to PNG and resized to fit 64 x 64. 64 px is enough for a map
marker and keeps the files tiny.

### SPEC-0023-REQ-005: Missing textures

If an icon texture is not found in the game files, then the icon script shall print
`nms-icons: <name> not found in the game files` to stderr, skip it, and continue; it ends with
`wrote <n> icons to <dir>`.

If any glyph texture is not found, then the glyph script shall print the missing digit and exit
non-zero: a glyph set with a hole would show a wrong address, which is worse than none.

### SPEC-0023-REQ-006: Serving

The exporter shall serve `/icon/<name>.png` only for names made of letters, digits, `.` and `_`
with no `..`, and `/glyph/<h>.png` only for a single hex digit (case-insensitive), reading from the
configured directory. If the file is absent, then it shall answer 404 with a plain-text body naming
the script to run (`no icon (run nms-icons)`, `no glyph (run nms-glyphs)`).

### SPEC-0023-REQ-007: Never committed

The repository shall not contain any extracted icon, glyph, texture, font or model, and the
output directories shall be listed in `.gitignore` (SPEC-0032).

### SPEC-0023-REQ-008: Which icon for which marker

Where icons are available, the map shall use them as follows, and its own shape otherwise:

| Map thing | Icon |
|---|---|
| Survey hotspot, class unknown | `surveying.<type>` (type = minerals, gas, power) |
| Survey hotspot, class set by the player | `surveying.<type><class>` (class = a, b, c, s) |
| Resource deposit seen through the visor | `pickup.mineral` |
| Portal | `building.portal` |
| Base, settlement | `base`, `settlement` |
| Exocraft, starship, corvette, freighter | `vehicle`, `ship`, `corvette`, `freighter` |
| The player's live position | `playerlocation` |
| A player-placed marker | `custommarker` |

The hotspot type in the game's marker is one of minerals, gas or power; the class is never in the
marker (SPEC-0022-REQ-003).

### SPEC-0023-REQ-009: Re-run after updates

Extraction is a maintenance step, like the table extractor: run once after install and again after
a game update. The scripts are idempotent; re-running overwrites the PNGs with the current build's
art. Neither script runs at exporter start-up.

## Degraded behaviour

- **Not extracted:** the map falls back to its own drawn shapes for markers; the systems page shows
  the address as hex digits with glyph names. Nothing else changes.
- **A game update renames a texture:** that icon is skipped with a warning; the map uses its shape
  for that marker type only. Re-find with `nms-pak find 'hud/icons'`.
- **No DDS converter installed:** both scripts fail on the first conversion; nothing is written.
- **Wrong glyph order** would silently show wrong addresses: hence the fixed order above and the
  verification below.

## Verification

1. `nms-icons` prints `wrote 26 icons to .../data/icons` on build 25800958, and `data/icons/`
   holds `surveying.{minerals,gas,power}{,a,b,c,s}.png` plus the 11 marker icons.
2. `nms-glyphs` prints `wrote 16 glyphs to .../data/glyphs`.
3. Open `/glyph/0.png`: it shall be the glyph the game calls Sunset; `/glyph/f.png` shall be
   Atlas. Compare a portal address shown on the systems page against the in-game portal
   for the same system.
4. `/icon/a%2E%2Eb.png`-style names containing `..`, and `/glyph/g.png`, return 404.
5. Delete `data/icons/`: the map still renders hotspots and markers with its own shapes.
6. `git status --ignored` lists `data/icons/` and `data/glyphs/` as ignored, never as untracked.

## Lessons

See [SPEC-0039](../lessons/SPEC-0039-game-data-lessons.md). The relevant one: the hotspot
**class is not in the hotspot marker the game exposes**, so the class-lettered icons are only
useful once the player has told the map the class (read from the visor). The icon set is
extracted in full anyway so the map can switch from the bare icon to the lettered one.
