---
id: SPEC-0024
title: 'Base part footprints from placement scenes'
description: 'Derive an object-local bounding box for every placeable base object from the game''s placement scenes (collision shapes first, mesh AABBs and the parts table as fallbacks) so the map can draw bases at true size.'
type: snapshot
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0020]
supersedes: []
superseded_by: []
related_specs: [SPEC-0031, SPEC-0032, SPEC-0039, SPEC-0060, SPEC-0066]
screenshots: [base-footprints.png, base-footprints-wide.png, base-layout.png]
---

# SPEC-0024: Base part footprints

## Summary

The save lists every base object as an id, a position and two orientation vectors (`Up`, `At`);
it says nothing about size. To draw a base from above at true scale — floors tiling, walls as thin
strips, a generator as a 2.8 m square — the map needs each object's extent in its own frame. The
game has it: every entry in the base-building objects table names a **placement scene** (the ghost
model shown while placing), and scenes carry collision shapes and mesh bounding boxes. `nms-footprints`
reads those into `data/part_footprints.json`: numbers only, no geometry or art (SPEC-0032). About
1,750 of 1,893 placeable objects get a box on build 25800958.

## Inputs and outputs

**Inputs** (via `nms-pak get`, SPEC-0020):

- `metadata/reality/tables/basebuildingobjectstable.mbin` — 2,124 `GcBaseBuildingEntry`
  entries; 1,893 have a non-empty `PlacementScene` (`TkModelResource` > `Filename`), naming
  1,040 distinct scenes.
- `metadata/reality/tables/basebuildingpartstable.mbin` — `GcBaseBuildingPart` entries whose
  `_id` is `_<PART ID>`, each with `Model` (`TkModelResource` > `Filename`).
- Every scene those reference, followed up to three levels.

**Output:** `data/part_footprints.json`, compact JSON (separators `,` `:`):

```json
{"source": "basebuildingobjectstable PlacementScene collision shapes / mesh AABBs (nms-footprints)",
 "frame": "object local: x right, y up, z forward; metres; [minx, miny, minz, maxx, maxy, maxz]",
 "footprints": {"T_WALL": [-2.667, 0.0, -0.167, 2.667, 3.333, 0.167], "...": []},
 "via": {"T_WALL": "placement collision", "...": ""}}
```

Keys are object ids without the save's leading `^`. `via` is one of `placement collision`,
`part collision`, `part mesh`, `placement mesh`. Values are rounded to 3 decimals. The file is
about 150 KB.

## Requirements

### SPEC-0024-REQ-001: Object -> placement scene

The tool shall split the objects table MXML on `value="GcBaseBuildingEntry" _id="`, take the id up
to the next `"`, and match
`name="PlacementScene" value="TkModelResource">\s*<Property name="Filename" value="([^"]*)"`;
objects with an empty filename are ignored. Scene paths are lower-cased.

### SPEC-0024-REQ-002: Part -> model

The tool shall split the parts table on `value="GcBaseBuildingPart" _id="_`, take the part id, and
match `name="Model" value="TkModelResource">\s*<Property name="Filename" value="([^"]+)"` (the first
`Model`, i.e. the default style), lower-cased.

### SPEC-0024-REQ-003: Batched unpacking

The tool shall unpack scenes by calling `nms-pak get '^(<p1>|<p2>|...)$' --all` with regex-escaped
paths, **80 paths per call**, mapping each printed `.MXML` line back to its packed `.mbin` path
(the part after `/files/`, lower-cased, `.mxml` -> `.mbin`). Progress goes to stderr.

### SPEC-0024-REQ-004: Scene parsing

For one scene MXML the tool shall split on `value="TkSceneNodeData"` and, per node, read `Type`
and the node's attribute list as `{Name: Value}` pairs (each attribute is a `Name` property,
optionally an `AltID` property, then a `Value` property). It shall collect three things:

**Collision boxes** (`Type` `COLLISION`): read `TransX/Y/Z`, `RotX/Y/Z` (degrees),
`ScaleX/Y/Z` from the node; half-extents by attribute `TYPE`:

| `TYPE` | half-extents (x, y, z) |
|---|---|
| `Box` | `WIDTH/2, HEIGHT/2, DEPTH/2` |
| `Cylinder` | `RADIUS, HEIGHT/2, RADIUS` |
| `Capsule` | `RADIUS, HEIGHT/2 + RADIUS, RADIUS` |
| `Sphere` | `RADIUS, RADIUS, RADIUS` |
| `Mesh` (or anything else) | skipped: no size in the scene |

Multiply half-extents by scale, take the 8 corners of `[-h, h]`, rotate each by X then Y then Z
(right-handed, standard rotation matrices), take the axis-aligned min/max, and add the
translation. Only the node's own transform is applied (collision nodes sit directly under the
scene root; parents are not composed). A node with a missing or non-numeric size is skipped.

**Mesh boxes:** any node with `AABBMINX..Z` and `AABBMAXX..Z` attributes contributes
`[minx, miny, minz, maxx, maxy, maxz]`.

**References:** any node whose `SCENEGRAPH` attribute, upper-cased, starts with `MODELS` and does
not contain `SNAPPOINT` contributes a reference, normalised by replacing `\` with `/` and
lower-casing (scene files write `MODELS\PLANETS\...\X.SCENE.MBIN`).

### SPEC-0024-REQ-005: Tiny-box filter

The tool shall discard every collision and mesh box whose largest dimension is **0.6 m or less**
before taking unions. Such boxes are placement arrows, locators and snap triggers, not the object.

### SPEC-0024-REQ-006: Scene box

A scene's collision box and mesh box shall each be the union (componentwise min of mins, max of
maxes) of its surviving boxes of that kind, or none.

### SPEC-0024-REQ-007: Following references

The tool shall load scenes breadth-first: the placement scenes first, then every newly referenced
scene, for up to **3 rounds**, never loading a path twice. A scene's extent of a kind is its own
box if it has one, else the union of its references' extents of that kind, recursing to depth 3.

### SPEC-0024-REQ-008: Collision vs mesh

For a set of scenes, the tool shall take `c` = union of collision extents and `m` = union of mesh
extents, and choose `c` unless `m` exists and **`m`'s ground area (x span x z span) exceeds 4 x
`c`'s** — in which case the collision covers under a quarter of the object (a dome whose collision
is only its door frame) and `m` is used. With no `c`, use `m`; with neither, none. (SPEC-0031.)

### SPEC-0024-REQ-009: Assignment order

For each object, the tool shall:

1. If its placement scene yields a box chosen from **collision**, record it as
   `placement collision`.
2. Else, if the parts table has a part whose id equals the object id, or else parts whose ids
   start with `<object id>_` (e.g. `CUBEROOM_FLOOR`, `CUBEROOM_WALL`, ...), defer it: load those
   part models (same reference following) and choose between their collision and mesh as in
   REQ-008, recording `part collision` or `part mesh`; if that yields nothing, fall back to the
   placement scene's mesh box as `placement mesh`.
3. Else, if the placement scene yields a mesh box, record `placement mesh`.
4. Else, report it as without a box.

The tool shall print `wrote <n> footprints to <file>; <k> without a box`.

### SPEC-0024-REQ-010: Consumer contract

A consumer shall turn a box into a ground polygon per object: with `p` = Position, `u` =
normalised `Up`, `f` = normalise(`At` - (`At`·`u`)`u`), `r` = `u` x `f`, the corners are
`p + r*x + f*z` for `(x0,z0), (x1,z0), (x1,z1), (x0,z1)` and the top is `p + u*y1`. Objects are
drawn lowest first so floors tile and upper storeys overlay. Pipes and power lines are drawn as
segments from Position to Position + At, not as boxes. The consumer reloads the file when its
mtime changes. (Drawing: SPEC-0066.)

## Degraded behaviour

- **Objects table cannot be unpacked:** exit `could not unpack the objects table`; no file
  written.
- **A referenced scene missing from the index** (renamed or never shipped): treated as an empty
  scene; the object falls through to the next rule.
- **No footprints file:** the layout API carries `"footprints": false` and the map draws objects
  without shapes (SPEC-0066).
- **Object id absent from the file** (new part after an update, or one of the ~143 without a box):
  that object alone is drawn as a point.
- **Mesh collision only** (`TYPE` `Mesh`): no size in the scene, so the mesh AABB or part model is
  used.

## Verification

On build 25800958 the run reports 1,750 footprints and 143 without a box; `via` counts are
placement collision 1,414, part mesh 154, part collision 96, placement mesh 86. Spot values
(metres, `[minx, miny, minz, maxx, maxy, maxz]`):

| Object | Box | via | Reads as |
|---|---|---|---|
| `T_WALL` (also `W_`, `C_`, `M_WALL`) | -2.667, 0, -0.167, 2.667, 3.333, 0.167 | placement collision | wall 5.333 x 3.333 x 0.333 |
| `T_FLOOR` (also `W_`, `C_`, `M_FLOOR`) | -2.666, -0.167, -2.666, 2.666, 0.167, 2.666 | placement collision | floor tile 5.333 m square, 0.333 thick |
| `CUBEROOM` | -4, 0, -4, 4, 4.043, 4 | part collision | cuboid room 8 m square |
| `CUBEFLOOR` | -2, -0.372, -2, 2, 0.025, 2 | part mesh | 4 m square |
| `U_GENERATOR_S` | -1.379, 0, -1.379, 1.379, 7.166, 1.379 | placement collision | Box 2.757 x 7.166 x 2.757 |
| `U_SILO_S` | -1.354, 0, -1.354, 1.354, 6.416, 1.354 | part collision | depot 2.71 m square |
| `U_EXTRACTOR_S` | -1.385, -0.53, -1.421, 1.385, 6.743, 1.644 | part mesh | |
| `U_BATTERY_S` | -0.677, 0, -0.486, 0.677, 1.754, 0.486 | placement collision | |
| `BIOROOM` | -6.18, -0.298, -6.18, 6.18, 4.256, 6.18 | part mesh | dome 12.36 m across |
| `BUILDLANDINGPAD` | -14, -0.812, -14, 14, 1.119, 14 | part mesh | 28 m pad |

Then look at it: on the map's base view a wall-and-floor building shall show floors tiling without
gaps or overlaps and walls on the tile edges; a generator shall not be a speck.

## Lessons

See [SPEC-0039](../lessons/SPEC-0039-game-data-lessons.md). Top two:

- **The first run boxed 526 of ~1,800 objects**, because placement scenes mostly *reference* other
  scenes and the references are upper-case backslash paths that never matched the lower-case
  forward-slash index. Normalise reference paths, and follow them.
- **Generators came out as a 0.2 m box** from the placement arrow mesh until collision shapes were
  read and boxes under 0.6 m were dropped.
