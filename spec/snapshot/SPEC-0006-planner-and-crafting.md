---
id: SPEC-0006
title: 'Planner and crafting: what the stock makes now and after a harvest'
description: 'A device planner for the high-value targets (craftable now / after one harvest, with the blocking ingredient), and a per-item crafting evaluation over every blueprint, refiner and cooking recipe.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0002, SPEC-0007]
supersedes: []
superseded_by: []
related_specs: [SPEC-0008, SPEC-0009, SPEC-0019, SPEC-0021]
screenshots: [conky-bases.png]
---

# SPEC-0006: Planner and crafting: what the stock makes now and after a harvest

## Summary

The question asked of every save is "how many Fusion Ignitors / Stasis Devices can I build right
now, what stops the next one, and what will the next harvest add?". The **planner** answers that
for a short list of targets, cheaply, on every save. The **crafting evaluation** answers the
general version for every item in the game (how many can be made now, through which route, what
raw materials one unit needs, the exact shortfall) and feeds the crafting dashboard; it takes
seconds, so it runs in the background.

## Inputs and outputs

- Stock: `{item_id: total}` across every container (SPEC-0007).
- Known blueprints: `KnownProducts` (ids, `^` stripped); known refiner recipes:
  `KnownRefinerRecipes`.
- Tables: `items[id].craft = {ingredients: [{id, amount}], yield}`, `items[id].value` (base sale
  value in units), `recipes[]` (refiner and cooking), `crops[part_id] = {yield_id, amount,
  grow_seconds}` (e.g. `RADIOPLANT -> PLANT_RADIO x 50, 14,400 s`).
- Planted crops: the `plants` section (SPEC-0007).

Outputs: `/json planner` and `/json crafting`, and the crafting metric families (SPEC-0002).

## Requirements

### SPEC-0006-REQ-001: Planner targets

The planner targets shall default to `ULTRAPROD1` (Fusion Ignitor) and `ULTRAPROD2` (Stasis
Device), overridable with a comma list in `NMS_PLAN_TARGETS`; ids not in the tables are dropped.

### SPEC-0006-REQ-002: Planner make step

`make(item, n)` on a mutable stock copy, for up to `n` units, one craft at a time:

- Only **known** blueprints are used (`item in KnownProducts` and has `craft`).
- For each ingredient short of `need`, if depth < **6** and the ingredient has a known blueprint,
  recursively make the difference first.
- If still short, record the first such ingredient as the blocker
  `{id, item, have, need}`, roll the stock back to before this craft, and stop.
- Otherwise subtract ingredients, add `yield` of the item.

Intermediates are crafted, never refined, in the planner.

### SPEC-0006-REQ-003: Planner run and order

For an order of targets, make up to **200** of each in turn on one shared stock copy; then, per
target, try one more unit on a copy of what is left to find the blocker of the *next* unit.
Value = Σ count × `items[t].value`. Both targets sell for the same base value, so the planner
shall run both orders (as given and reversed) and keep the higher value.

### SPEC-0006-REQ-004: After one harvest

`harvest_adds` = for every planted crop, `crops[plant.id].amount` of `crops[plant.id].yield_id`,
summed. The planner shall repeat REQ-003 on `stock + harvest_adds` and report:

```
planner = {
  targets: [{id, item, value, now, after_harvest, blocker, blocker_after_harvest}],   # each target alone
  best_now: {order, counts, names, blockers, value},
  best_after_harvest: {...},
  harvest_adds: {item name: amount}    # largest first
}
```

The planner runs synchronously in the snapshot build; an exception sets `errors.planner` and
`planner = {}`.

### SPEC-0006-REQ-005: Crafting routes

Every item shall get routes from three sources: `craft` (its blueprint; known iff in
`KnownProducts`), `refine` and `cook` (each recipe whose result it is; known iff in
`KnownRefinerRecipes`). Refiners and nutrient processors run any recipe whether discovered or not,
so with "known only" on, refine and cook routes are always usable and craft routes need the
blueprint.

### SPEC-0006-REQ-006: Route choice

For an item in a given expansion, skip routes with an ingredient already on the expansion stack
(loop guard: recipes that need their own output as a seed). Pick the route with the largest
`direct` count from the preference inventory, then craft over refine/cook, then fewest
ingredients. Items with no usable route are leaves (gathered).

### SPEC-0006-REQ-007: Expansion and makeable

`expand(item, qty)`: take what stock has; for the rest, if depth < 6 and a route exists, do
`ceil(qty / yield)` crafts and expand each ingredient × crafts; else add to shortfall.
`makeable(item)` = the largest n (exponential search then bisection, cap 1,000,000) such that
expanding n on a fresh stock copy has no shortfall. Because the simulation consumes a copy of
the inventory, shared bottlenecks (chromatic metal feeding two branches) count once.

### SPEC-0006-REQ-008: Per-item results

For each item with any route: `known`, `routes[{route, recipe, recipe_id, yield, needs,
known, direct}]` (`direct` = one-level count from stock), `best_route`, `best_recipe`,
`makeable`, `raw_need` (expand 1 unit against an empty inventory, routes chosen by what is held),
`shortfall` (expand 1 unit against stock).

### SPEC-0006-REQ-009: Background evaluation

The evaluation shall run on a thread after each new snapshot, store the result for metrics, and
replace the snapshot's `crafting` value with a summary: known blueprints (craft route known) not
makeable, with their `shortfall`, sorted by (kinds missing, units missing, name), top 150; and
`by_ingredient` groups `{id, ingredient, have, items[{item, short, only}]}` sorted by number of
blocked items, top 24. `crafting.computed_at` is the Unix time it finished.

## Degraded behaviour

- **Before the first evaluation finishes**: `crafting.computed_at = null`, empty lists, crafting
  metrics absent. The planner is already there (it runs in the build).
- **Evaluation fails**: `errors.crafting` set; the previous result's metrics stay.
- **Tables missing**: planner targets list is empty; the screen says "no planner data".
- **Nothing planted**: "after harvest" equals "now".

## Verification

1. With stock for exactly one Fusion Ignitor (check its tree in game), `planner.targets[0].now`
   is 1 and the blocker names the ingredient that runs out next.
2. Plant a crop that unblocks a target; `after_harvest > now` and `harvest_adds` lists the crop.
3. Crafting dashboard: pick an item you can craft once; "can make" is 1; its shortfall for one
   more unit matches what the game's crafting menu says is missing.
4. `nms_crafting_compute_seconds` stays well below the scrape interval (a few seconds).

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): substances with a refiner recipe
(Star Bulb, Cactus Flesh, Solanium, Gamma Root, Faecium, Fungal Mould) must expand through it,
or "blocked" is overstated; refine/cook recipes need no discovery.
