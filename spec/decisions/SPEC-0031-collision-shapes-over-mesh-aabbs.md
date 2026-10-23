---
id: SPEC-0031
title: 'Part footprints: collision shapes first, mesh bounding boxes as a fallback'
description: 'A base object''s footprint comes from its placement scene''s collision shapes; mesh AABBs are used only when there is no collision or it covers under a quarter of the mesh''s ground area, and boxes under 0.6 m are discarded.'
type: decision
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0024, SPEC-0039]
---

# SPEC-0031: Collision shapes over mesh AABBs

## Context

The map draws each base object as a rectangle on the ground, from an object-local box
(SPEC-0024). Scene files offer two sources of size:

- **Mesh AABBs** — every mesh node carries `AABBMIN/MAX X/Y/Z`. Present almost everywhere, but a
  placement scene's meshes include the things shown *while placing*: arrows, locators, snap-point
  markers. The first implementation used only the union of mesh AABBs, documented as "the union of
  those is the object's extent". The electromagnetic generator came out as a **0.2 m box** — the
  placement arrow — because its real model lives in a referenced scene and its placement scene's
  only own mesh is the arrow. Other utility parts showed the same failure in smaller ways.
- **Collision shapes** — `COLLISION` nodes with `TYPE` Box (`WIDTH/HEIGHT/DEPTH`), Cylinder or
  Capsule (`RADIUS/HEIGHT`) or Sphere (`RADIUS`), positioned by the node's translation, rotation
  and scale. This is the solid the game itself uses; the generator's is a single Box
  2.757 x 7.166 x 2.757 m. But some objects have only a token collision — a dome whose collision
  is its door frame — and `Mesh` collision carries no size in the scene.

## Decision

1. Read collision shapes and apply the node's own transform; ignore `Mesh`-type collision.
2. Discard every box (collision or mesh) whose largest side is **0.6 m or less**: arrows, locators
   and snap triggers, never a real base part.
3. Per object, prefer the collision union; use the mesh union instead **only when the mesh's ground
   area (x span x z span) exceeds 4 x the collision's** (collision covers under a quarter of the
   object), or there is no collision.
4. If the placement scene gives no collision, try the parts table models (`<ID>` or `<ID>_*`)
   with the same rule before falling back to the placement mesh.
5. Record which rule produced each box (`via`) so a wrong footprint can be traced.

On build 25800958: 1,414 placement collision, 96 part collision, 154 part mesh, 86 placement mesh,
143 none, of 1,893 objects with a placement scene.

## Alternatives considered

- **Mesh AABBs only** — the original approach; wrong for generators and anything whose placement
  scene is mostly helpers.
- **Collision only** — wrong for domes and other token-collision objects, and leaves `Mesh`
  collision objects without a size.
- **Full geometry (read the mesh vertex data)** — exact, but means parsing geometry streams and
  storing derived game geometry; a rectangle is all the map needs, and it would edge toward
  redistributing art (SPEC-0032).
- **Hand-maintained sizes for common parts** — fine for walls and floors, hopeless for ~1,900
  objects across updates.

## Consequences

- Footprints are the game's own collision boxes, so walls, floors and generators measure exactly
  (wall 5.333 x 3.333 x 0.333 m, floor tile 5.333 m square, cuboid room 8 m square, generator
  2.757 m square).
- Rotated collision boxes are reduced to their axis-aligned bounds in object space; a 45-degree
  box overstates its footprint by up to 41 %. Acceptable for a map.
- Node transforms are taken one level deep; a scene with collision nested under transformed
  parents would be misplaced. Not seen on build 25800958; revisit if a footprint is visibly offset.
- The 0.6 m and 4x thresholds are empirical. If an update adds genuinely small base parts (under
  0.6 m on every side), they will lose their footprint and fall back to a point; lower the
  threshold and check the arrow cases again.
