---
id: SPEC-0025
title: 'Harvest All radius: a loose-file mod built from the installed game'
description: 'Install, inspect and remove a one-value mod that raises the freighter Hydroponic Room''s Harvest All radius, rebuilt from the user''s own game files rather than shipped.'
type: snapshot
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0020]
supersedes: []
superseded_by: []
related_specs: [SPEC-0022, SPEC-0026, SPEC-0032, SPEC-0039]
screenshots: []
---

# SPEC-0025: Harvest All radius mod

## Summary

"Harvest All Plants" in a freighter Hydroponic Room only reaches plants within 6 m, so a large
freighter farm needs one interaction per room. The radius is a single data value on the room's
floor entity. `nms-mod-harvest-all` unpacks that entity from the user's installed game, changes
only that value, recompiles it, and installs it as a **loose-file mod** under `GAMEDATA/MODS/`,
with a marker file recording what it did and for which build. It refuses to touch anything while
the game runs, refuses if the stock file is not what it expects, and `remove` restores the game
completely. No game file is shipped: the mod is rebuilt from the installed copy (SPEC-0032).

## Inputs and outputs

**Inputs:**

- `models/planets/biomes/common/buildings/parts/buildableparts/freighterbase/rooms/plantroom/parts/floor0/entities/interaction.entity.mbin`
  in `NMSARC.Precache.pak`, via `nms-pak get` (SPEC-0020).
- MBINCompiler and a .NET 8 runtime from the tools directory (MXML -> MBIN).
- Game directory: default `<steamapps>/common/No Man's Sky`, override `NMS_GAME_DIR`.
- Steam manifest `<steamapps>/appmanifest_275850.acf` (two levels above the game directory) for
  the build id.
- `/proc/*/comm` to detect the running game (process name `NMS.exe`, under Proton/Wine).

**Outputs:**

- `<game>/GAMEDATA/MODS/HarvestAllFreighter/<PACKED PATH, UPPER CASE, / as directory separators>`
  — the recompiled MBIN, i.e.
  `.../MODS/HarvestAllFreighter/MODELS/PLANETS/BIOMES/COMMON/BUILDINGS/PARTS/BUILDABLEPARTS/FREIGHTERBASE/ROOMS/PLANTROOM/PARTS/FLOOR0/ENTITIES/INTERACTION.ENTITY.MBIN`.
- `<game>/GAMEDATA/MODS/HarvestAllFreighter/nms-exporter-mod.json`:
  `{"what": "Freighter Hydroponic Room Harvest All radius", "radius": 300.0, "stock_radius": 6.0,
  "file": "<packed path>", "game_build": "<build id>", "made_by": "<tool name>"}`.

**CLI:** `install [--radius R]` (default 300), `status`, `remove`.

## Requirements

### SPEC-0025-REQ-001: The field

The tool shall change exactly one value: in the floor entity MXML, the `Radius` property that
immediately follows `<Property name="GcHarvestPlantAction">`, matched by

```
(<Property name="GcHarvestPlantAction">\s*<Property name="Radius" value=")([0-9.]+)(" />)
```

This action sits in a `GcTriggerActionComponentData`, state `HARVEST`, reached from a
`GcSimpleInteractionComponentData` named `UI_HARVEST_ALL` ("Harvest All Plants") with
`TriggerAction` `HARVEST`. The new value is written with six decimals (`300.000000`).

### SPEC-0025-REQ-002: Stock check

If the pattern matches other than exactly once, then the tool shall refuse
(`expected one GcHarvestPlantAction Radius ..., found <n>: the file changed; not installing`).
If the matched value is not 6.0 (within 1e-6), then the tool shall refuse
(`stock Radius is <v>, not 6.0: the game changed it; check before installing`).
Either means a game update changed the file and a human should look before overriding it.

### SPEC-0025-REQ-003: Radius bounds

The tool shall accept `--radius` only in **6 to 2000** inclusive.

### SPEC-0025-REQ-004: Refuse while running

While a process whose `comm` is `NMS.exe` exists, the tool shall refuse `install` and `remove`
(`No Man's Sky is running; quit it first (mods load at start-up).`). The game reads loose-file
mods only at start-up, and replacing a file it has open helps no one.

### SPEC-0025-REQ-005: Build

When installing, the tool shall write the modified MXML to a scratch directory beside the unpacked
original (`<dir>/harvest-all-mod/interaction.entity.MXML`), run
`MBINCompiler-linux convert -y --force -q <file>` there with `DOTNET_ROOT=<tools>/dotnet8`, and
require `interaction.entity.MBIN` to exist afterwards; otherwise refuse with MBINCompiler's output.

### SPEC-0025-REQ-006: Install layout

The tool shall copy the MBIN to `GAMEDATA/MODS/HarvestAllFreighter/` + the packed path upper-cased
with each `/` a directory, creating directories, then write `nms-exporter-mod.json` with the
current build id, and print the destination and a reminder that the game shows a "mods installed"
notice on start-up.

### SPEC-0025-REQ-007: Status

`status` shall print `not installed` if the mod folder is absent; say the folder is not this
tool's if the marker JSON is missing or unreadable; else print the radius and build it was made
for and, **if the installed build differs, a warning to reinstall** (which rebuilds from the new
game files) or remove.

### SPEC-0025-REQ-008: Remove

`remove` shall refuse if the game is running, print `not installed` if absent, **refuse to delete a
folder without its marker JSON** (it is someone else's), else delete the mod folder and then
`GAMEDATA/MODS/` itself if that is now empty.

### SPEC-0025-REQ-009: Never touch the paks

The tool shall never write inside `PCBANKS` or modify any packed file. Removing the mod folder
returns the game to stock.

## Degraded behaviour

- **Game update changed the entity:** stock check refuses; nothing installed. Look with
  `nms-pak grep GcHarvestPlantAction --in plantroom`, update the expectation, retry.
- **Game update without changes to the entity:** the old mod still loads; `status` warns about the
  build mismatch. Reinstall to be safe; the loose file shadows the packed one, so an updated
  packed file would otherwise be hidden.
- **Another mod ships the same file:** they conflict; the game loads one. Not detected.
- **MBINCompiler version mismatch:** conversion fails or produces a file the game rejects; the
  tool refuses on a missing output, but cannot detect a subtly wrong one (see Verification 3).
- **The game's behaviour with large radii is not established:** whether it is a sphere or a
  cylinder, whether it crosses floors, and its performance cost are open questions (SPEC-0026
  question register). The mod is a test as much as a feature.

## Verification

1. `install --radius 300` with the game closed prints the destination; `status` prints
   `installed: radius 300.0 for game build <id>`.
2. Round trip: the original reported its recompiled MBIN differing from the packed one only in
   MBINCompiler's 5 header bytes (apart from the edited value). A larger difference means an
   MBINCompiler built for another game version.
3. Decompile the installed MBIN with MBINCompiler: the MXML equals the stock MXML except
   `Radius` `300.000000`.
4. With the game running, `install` and `remove` both refuse.
5. Start the game: it shows a "mods installed" notice. In a freighter with hydroponic rooms on two
   floors, use Harvest All once and check which plants were harvested (the next save's plant
   timestamps show it). The original's farm: 1,549 plants over 156 x 163 m on two floors 8 m
   apart; the farthest plant from any planter slot is 223 m, so 300 covers the farm from anywhere
   in it.
6. `remove` deletes the folder; the game starts without the mods notice.

## Lessons

See [SPEC-0039](../lessons/SPEC-0039-game-data-lessons.md). The one that shaped this tool: the radius
was first declared **hardcoded** after checking only the globals and the crop's entity. It is on the
*room's floor entity* (the crop entity has only `ScanRange` 550 and
`DeactivateSimilarInteractionsNearbyRadius` 0). Search packed paths for the object the player
interacts with, not the object being acted on.
