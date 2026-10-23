---
id: SPEC-0039
title: 'game-data lessons'
description: 'What unpacking, reading and modding the game''s own files cost to learn: wrong paths, wrong boxes, wrong divisors, and answers given before the evidence was in.'
type: lesson
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
related_specs: [SPEC-0020, SPEC-0021, SPEC-0022, SPEC-0023, SPEC-0024, SPEC-0025, SPEC-0026, SPEC-0030, SPEC-0031, SPEC-0032]
---

# SPEC-0039: game-data lessons

Ordered by how much time each would save a second builder.

## Lesson: scene references are upper-case backslash paths — normalise and follow them

**What happened.** The first footprint run boxed **526 of about 1,800** placeable objects. Most
placement scenes hold little geometry of their own; they reference other scenes through a
`SCENEGRAPH` attribute written as `MODELS\PLANETS\BIOMES\...\X.SCENE.MBIN`. The pak index lists
packed paths lower-case with forward slashes, so every reference silently failed to resolve and
those objects fell through to "no box". Nothing errored; the count was just low.

**The rule.** Lower-case every path read out of a game file and replace `\` with `/` before looking
it up. Follow references (three levels was enough on build 25800958, giving 1,750 of 1,893). Print
coverage (`wrote N footprints; K without a box`) on every run and treat a sudden drop as a bug.

## Lesson: the placement scene's own mesh is often the placement arrow

**What happened.** With only mesh bounding boxes, the electromagnetic generator came out as a
**0.2 m box**: its placement scene's only mesh is the arrow shown while placing it, and its real
model is a reference. Utility parts with locators and snap triggers were similarly shrunk or
shifted. The scene does carry a `COLLISION` node — a Box 2.757 x 7.166 x 2.757 m — which is what
the game itself treats as the solid.

**The rule.** Use collision shapes (Box / Cylinder / Capsule / Sphere with the node's transform)
first; drop any box with no side over 0.6 m; fall back to mesh AABBs only when there is no
collision or it covers under a quarter of the mesh's ground area (a dome whose collision is its
door frame). Record which rule produced each box. Then *look at a drawn base* — walls 5.333 m long
and 0.333 m thick on the edges of 5.333 m floor tiles — because a plausible-looking JSON file
proved nothing (SPEC-0031).

## Lesson: "hardcoded" is a hypothesis until a census says otherwise

**What happened.** Asked how far Harvest All reaches, an early session searched the gameplay globals
and the crop's interaction entity, found no radius, and answered that it was hardcoded. It is a data
value — `GcHarvestPlantAction.Radius` 6.0 — on the **room's floor entity**
(`.../freighterbase/rooms/plantroom/parts/floor0/entities/interaction.entity.mbin`) and on the
Biodome's entity. The crop entity has only `ScanRange` 550 and
`DeactivateSimilarInteractionsNearbyRadius` 0. The wrong answer would have ruled out a one-value
mod that now exists (SPEC-0025).

**The rule.** Settings live on the object the player interacts with, not the object acted on.
Search packed *paths* first (`find` over ~195,000 paths is instant), unpack candidates, then grep
contents. Never say "not in the data" without the stored `find`/`grep` that would have found it
(SPEC-0026-REQ-007).

## Lesson: refiner `TimeToMake` is not seconds

**What happened.** Recipe `TimeToMake` read as seconds made every refiner ETA wrong. The game
divides it by `RefinerSubsMadeInTime` (**250**) when the output is a substance and by
`RefinerProductsMadeInTime` (**2**) when it is a product (survival mode 100 / 1). One op consumes
the listed ingredients and yields `result.amount`.

**The rule.** Derive seconds per op from the globals, keep both raw and derived values in the
tables with a `units` note, and check a projected ETA against two saves of one refiner before
trusting it.

## Lesson: a refiner stalls at the product stack size

**What happened.** Gas -> product recipes (Enriched Carbon, Nitrogen Salt, Thermic Condensate)
were projected as if they could run until their inputs ran out. They stop when the output slot is
full: 9,999 for substances but
**10 for products**, so they stall in minutes. No single table says this.

**The rule.** Cap projected refiner output at the result's stack size; for product outputs, prefer
crafting (250 gas + 50 Condensed Carbon) over refining in plans.

## Lesson: hgpaktool `-L` writes `filenames.json` into the current directory

**What happened.** Listing pak contents with `hgpaktool -L` writes its `filenames.json` into
whatever directory the command was run from, ignoring `-O`. It turned up as an untracked file in an
unrelated repository that happened to be the working directory.

**The rule.** Run `-L` from a fresh temporary directory, read the JSON, delete the directory. Do
the same for any tool whose output location you have not verified.

## Lesson: keep the pak toolchain somewhere persistent and writable

**What happened.** The first toolchain (hgpaktool, MBINCompiler, a .NET 8 runtime) was unpacked
under a temporary directory, so it was lost with the temporary directory and had to be fetched
again. When it moved to a system location, ownership mattered: MBINCompiler writes a log file
beside its own binary, so the directory has to be writable by the user who runs the conversion.

**The rule.** One tools directory, persistent, owned by the user who runs the scripts; every script
takes its path from one environment variable with that directory as default. Pin the MBINCompiler
release to the game version (one release per game version) and record its version in the tables'
`meta`.

## Lesson: data is not behaviour — storms

**What happened.** `gcskyglobals` gives storm lengths by planet storm frequency: Low 120–180 s,
High 150–320 s. The first countdown took the planet's StormFrequency (Low) at face value. The measured
storm lasted **315 s**, and the next arrived 359 s after the first cleared (Low predicts
900–2,400 s). The planet was Extreme: it uses the High length and the ExtremeFallback gap
(240–620 s) whatever its StormFrequency.

**The rule.** A table value says how the game is configured, not which branch the code takes.
Mark a concept `game-data` for the value and add a `measured` section before a UI predicts from
it; show predictions as estimates until then.

## Lesson: what the player most wants is not in the save or the marker

**What happened.** Hotspot class was looked for in two places and found in neither: it is not in the save (the
save keeps nothing about hotspots), and not in the live hotspot marker either — a class B and a
class S marker differ only in position, label and marker type. Likewise a mineral hotspot can give
any of the planet's three listed resources, and the marker does not say which. For power: the
class of the hotspot under a generator, biofuel fuel (`UserData` is 256 on every biofuel
generator), and the wire graph (saved wires stop 4–15 m short of generators) are all absent.

**The rule.** Ask the player and store their answer (SPEC-0072). Compute power verdicts from what
is known (extractor demand vs rated supply) and present the rest as "up to".

## Lesson: version laundering from wikis

**What happened.** Community pages give network thresholds of 1,000–2,500 items/h with no method
or version, and pre-update numbers circulate as current. The main wiki answered HTTP 402 to every
automated fetch, so its claims could not even be stored with a date.

**The rule.** Every claim names its build (game data, measurements) or its page date and patch
(community). Unstorable sources are not citable. Prefer the installed build's files (SPEC-0030).

## Lesson: small format traps in the tables

**What happened.** Each of these cost a debugging round:

- `RegionHotspotSubstances` spells the field **`SubstanceYeild`**.
- Procedural products' names are templates containing `%...%`; fall back to the `Subtitle` string.
- The save's `MissionProgress.Progress` is a progress number, not a stage index; stage numbers
  come from each stage's `Versions` table at the highest version.
- Some language files exist in two English variants; read `nms_*_english` and skip `usenglish`.
- Enum fields wrap their value in a child property named for the enum (`InventoryType`,
  `LinkNetworkType`); reading the outer property's `value` gives the wrapper, not the enum.

**The rule.** When a field reads empty or absent, open the MXML around it (`nms-pak show --lines`)
before assuming the data is missing.

## Lesson: check the stock value before overriding a game file

**What happened.** The Harvest All mod could have been a blind find-and-replace. Instead it requires
exactly one `GcHarvestPlantAction` `Radius` and that it be the stock 6.0, and records the build it
was made for. A game update that changes the file then stops the install rather than shipping a
stale entity that shadows the new one.

**The rule.** A loose-file mod built from game files must assert the stock state it expects,
refuse while the game runs (mods load at start-up), record the build, and warn when the installed
build moves on.
