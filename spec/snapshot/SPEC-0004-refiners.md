---
id: SPEC-0004
title: 'Refiners: recipe matching and where each refiner is'
description: 'Read RefinerBufferKeys/RefinerBufferData, keep the newest entry per position, infer the recipe from the slots, and name the base, freighter or corvette the refiner is in.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0001, SPEC-0002]
supersedes: []
superseded_by: []
related_specs: [SPEC-0003, SPEC-0007, SPEC-0019]
screenshots: [conky-bases.png]
---

# SPEC-0004: Refiners: recipe matching and where each refiner is

## Summary

The save does not store "this refiner is making Gold". It stores, for every refiner-like machine
the player has used, a key (a position and an integer location code) and a small inventory of
slots with item ids and amounts. The exporter has to (1) pick the newest record per machine,
(2) throw away records that are not refiners, (3) work out which slot is the output and which
recipe produced it, and (4) say where the machine is in words the player recognises ("Cobalt
Oxygen", "Freighter", "Corvette"). The recipe then gives the time per operation, which is what
the projection (SPEC-0003) runs on.

## Inputs and outputs

From `PlayerStateData`:

| Field | Shape |
|---|---|
| `RefinerBufferKeys[i]` | `GcMaintenanceSaveKey`: `{Location: int, Position: [x,y,z]}` |
| `RefinerBufferData[i]` | `{LastUpdateTimestamp, LastCompletedTimestamp, InventoryContainer: {Slots: [...]}}` |
| `PersistentPlayerBases[]` | `{Name, BaseType.PersistentBaseTypes, Position, Forward, Objects[]}` |
| `PlayerFreighterName` | freighter display name |

From the tables (`game-data` area): `recipes[]` with `id`, `operation` (display name),
`cooking` (bool), `seconds_per_op`, `result {id, amount}`, `ingredients [{id, amount}]`, and
`items[id].type` (`Substance` / `Product`).

`seconds_per_op = TimeToMake / 250` for substance results and `/ 2` for product results (the
game's `RefinerSubsMadeInTime` / `RefinerProductsMadeInTime`; survival uses 100 and 1).

Output row (`/json refiners[]`): `key` (`"<loc>:<x>,<y>,<z>"`, rounded ints), `location_code`,
`base`, `recipe`, `recipe_id`, `seconds_per_op`, `output_id`, `output`, `output_amount`,
`output_per_op`, `output_cap`, `inputs [{id,item,amount,per_op}]`, `ops_remaining`,
`last_update`, `last_completed`, plus the projected fields from SPEC-0003.

## Requirements

### SPEC-0004-REQ-001: Newest record per machine

The exporter shall zip keys with data, group by `(Location, rounded Position)` and keep the entry
with the largest `LastUpdateTimestamp`. The list holds history: the same machine appears many
times.

### SPEC-0004-REQ-002: Skip non-refiners

The same list carries extractor rooms, damaged machinery, drop pods and frigate repair slots. The
exporter shall skip an entry when any slot id is `^MAINT_HOOVER` (gas/mineral extractor room),
when it has no `Product` or `Substance` slots, when no recipe matches (REQ-003), or when its
`LastUpdateTimestamp` is more than 7 days old. Cooking recipes (`cooking: true`) are not refiner
candidates.

### SPEC-0004-REQ-003: Recipe matching

For each slot `s` taken as a candidate output `out` and each non-cooking recipe whose result is
`out`, with `need` = the recipe's ingredient ids and `others` = the other slot ids:

- the recipe is a candidate only if `need ⊆ others` or `need ⊆ all slot ids`;
- score = `20` if `s.Amount == -1` (the game's "output chosen, nothing made yet" placeholder)
  `+ 10` if `need == others` `+ 5` if `need ⊆ others` `+ 1` if `out ∉ need`;
- the highest score wins.

The `+1` handles recipes whose output is also an input (Oxygenate Carbon): they win only when
nothing else fits.

### SPEC-0004-REQ-004: Amounts

Input amounts shall be `max(0, slot.Amount)` (0 for a missing slot). `ops_remaining = min over
ingredients of amount // per_op`. `output_amount = max(0, out slot Amount)`.
`output_cap = 9999` for a Substance output, else the slot's `MaxAmount`.

### SPEC-0004-REQ-005: Location codes

`Location` has no enum in the game's metadata. Read off a save:

| Code | Meaning | Name shown |
|---|---|---|
| 1 | Planet surface outside a base (planet coordinates, no address) | `Planet surface` |
| 2 | Freighter | `Freighter <PlayerFreighterName>` |
| 3 | Planetary base | the base's `Name` when matched (REQ-006), else `Planet base` |
| 5 | Space | `Space` |
| 6 | Frigates | `Frigate` |
| 8 | Corvette (keys land on its `B_WALL_TECH0` modules) | `Corvette` |

Other codes shall render as `location <n>`.

### SPEC-0004-REQ-006: Which planetary base

Location-3 positions are relative to the base origin but on **planet-aligned axes**, not the
base's own. For each planetary base (skip `FreighterBase` and `PlayerShipBase`):

```
up  = normalise(base.Position)                       # from the planet centre
f   = normalise(Forward - (Forward·up) * up)         # base Forward made orthogonal
r   = normalise(up × f)
for each object whose ObjectID contains "REFINER":
    p = object.Position                               # base-local
    world_offset = r*p.x + up*p.y + f*p.z
```

The refiner's key position is matched to the nearest such offset; if within **2.0** units, the
row takes that base's name.

### SPEC-0004-REQ-007: Base display names

`FreighterBase` shall be named `Freighter <PlayerFreighterName>` and `PlayerShipBase` `Corvette`;
other bases use their `Name` (falling back to the base type, then `base`).

## Degraded behaviour

- **No recipe tables**: no refiner matches; `refiners` is `[]` and the screens say "no refiners
  in the save". Nothing else is affected.
- **Recipe changed by a game update**: the entry stops matching and disappears rather than showing
  a wrong recipe; regenerate the tables.
- **Base not matched** (moved base, odd frame): the row says `Planet base` instead of a name.
- **Ambiguous slots** (two recipes fit): the score picks deterministically; a wrong pick shows a
  plausible but wrong recipe. Verify against the game when adding recipes.

## Verification

1. In game, note each running refiner's output, input and location. `/json refiners` lists the
   same set with the same output names and the right base/freighter/corvette.
2. Put an empty output in a fresh refiner (`Amount == -1` case) and save: it matches the chosen
   recipe, not a coincidental one.
3. Find a recipe in the tables whose result is also one of its ingredients (the game's
   "Oxygenate Carbon" operation is one); load a refiner with the ordinary recipe for that output
   and check the ordinary recipe wins, not the self-feeding one.
4. Time one operation in game against `seconds_per_op` (normal mode) before trusting ETAs.

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): the location frame for base
refiners is planet-aligned, not base-aligned; matching in the base's own frame finds nothing.
