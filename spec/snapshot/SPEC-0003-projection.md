---
id: SPEC-0003
title: 'Projecting the save forward on the wall clock'
description: 'Refiners, extraction farms, plants and frigate expeditions are advanced from the save''s own timestamps to the request time at the game''s rates, so numbers keep moving between autosaves.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0001, SPEC-0002]
supersedes: []
superseded_by: []
related_specs: [SPEC-0004, SPEC-0005, SPEC-0007, SPEC-0013]
screenshots: [conky-bases.png]
---

# SPEC-0003: Projecting the save forward on the wall clock

## Summary

The game writes a save every few minutes while playing and never while closed, so a raw reading
is a photograph that ages. Everything with a known rate is therefore **projected**: each row
carries an *anchor* (the Unix time the game stamped on that object in the save) and the amounts at
that time; on every request the exporter computes what the amounts are now, capped at the
object's capacity. A new save replaces the anchors with the truth and the projection restarts from
there. The reasoning is in SPEC-0013. This spec gives the formulas for each kind of row, which are
small enough to state exactly.

## Inputs and outputs

Input: the static snapshot (SPEC-0002) and `now` (request wall-clock, Unix seconds). Output: the
same rows with the projected fields below added. `save.age = now - save.mtime` is added to every
response and exported as `nms_save_age_seconds`; every consumer shows it, because every timer
depends on it.

| Row | Anchor field (from the save) | Rate source |
|---|---|---|
| Refiner | `RefinerBufferData[i].LastUpdateTimestamp` | recipe `seconds_per_op` from the tables |
| Farm | max of the base's `LastUpdateTimestamp` and each member object's `Timestamp` | measured units/s, else 100 units/s per extractor (SPEC-0005) |
| Plant | the plant object's `Timestamp` | 1 s of growth per wall-clock second |
| Expedition | `FleetExpeditions[i].StartTime` | `events x event_seconds` (5400 s per event) |

## Requirements

### SPEC-0003-REQ-001: Refiner projection

For each refiner with `spo = seconds_per_op` (treat 0 as 1), `elapsed = max(0, now - last_update)`:

```
ops        = ops_remaining                                   # min over inputs of amount // per_op
if output_cap and output_per_op:
    ops_to_cap    = max(0, (output_cap - output_amount) // output_per_op)
    stalls_at_cap = ops_to_cap < ops
    ops           = min(ops, ops_to_cap)
done               = min(ops, floor(elapsed / spo))
projected_output   = min(output_cap or inf, output_amount + done * output_per_op)
projected_inputs[i]= max(0, input[i].amount - done * input[i].per_op)
remaining_seconds  = max(0, ops * spo - elapsed)
finish_at          = last_update + ops * spo
progress           = 1.0 if ops == 0 else min(1, done / ops)
active             = 1 if ops > 0 and remaining_seconds > 0 else 0
```

`output_cap` is 9,999 for a substance output and the slot's `MaxAmount` for a product. A refiner
whose output reaches the cap stalls with inputs left; `stalls_at_cap` lets a screen say "full -
empty it" instead of "done".

### SPEC-0003-REQ-002: Farm projection

For each farm row with `rate` in resource units/s, `upi = 1440` units per item:

```
elapsed          = max(0, now - anchor)
projected_units  = min(capacity_units, stored_units + rate * elapsed)
projected_items  = projected_units / upi
fill_ratio       = projected_units / capacity_units   (0 if capacity is 0)
full_in_seconds  = (capacity_units - projected_units) / rate   if rate > 0 else null
```

### SPEC-0003-REQ-003: Plant projection

For each plant with `grow_seconds` (the base part's PlantGrowth storage, e.g. 14,400 s for Gamma
Weed) and `grown` (upper 32 bits of `UserData`, seconds already grown):

```
projected_grown  = min(grow_seconds, grown + max(0, now - anchor))
ready_in_seconds = max(0, grow_seconds - projected_grown)
mature           = projected_grown >= grow_seconds
```

Per base: `total` plants and `mature` count.

### SPEC-0003-REQ-004: Expedition projection

```
eta               = start + events * event_seconds         (computed at build time)
remaining_seconds = max(0, eta - now)
progress          = min(1, (now - start) / (events * event_seconds))   (1.0 when events == 0)
returned          = 1 if remaining_seconds == 0 else 0
```

Event outcomes (`events_done`, `events_failed`, damaged frigates) are **not** projected: the game
only resolves them while running, so they are reported "as saved".

### SPEC-0003-REQ-005: Saved values stay available

Next to every projected value the exporter shall keep the saved one (`output_amount`,
`stored_items`, `nms_refiner_output_saved`, `nms_farm_stored_items_saved`) and the anchor
(`nms_refiner_last_update_seconds`, `nms_farm_anchor_seconds`), so a reader can always tell the
measured part from the extrapolated part.

### SPEC-0003-REQ-006: Stale entries are not projected

Refiner buffer entries whose `LastUpdateTimestamp` is more than **7 days** old shall be dropped
at build time (SPEC-0004); the save keeps entries for machinery the player dismantled long ago.

### SPEC-0003-REQ-007: Snap back on a new save

When a new save is decoded, every anchor and saved amount shall be replaced from it; nothing from
the previous projection carries over except the farm fill history used for measured rates.

## Degraded behaviour

- **Game closed for days**: projections run to their caps and stop (refiners finish or stall,
  farms fill, plants mature, expeditions return). `save.age` grows; the wall screens' save pill
  turns from green to idle grey after 15 minutes (SPEC-0009).
- **Wrong rate**: an assumed farm rate over- or under-shoots until two consecutive saves give a
  measured one (SPEC-0005). The row carries `rate_measured: false` and the screens say
  "assumed C-class".
- **Clock skew**: anchors are the game's Unix times written on the same machine, so skew only
  matters if the scraper's clock differs; `elapsed` is clamped at 0 so a future anchor never
  projects backwards.
- **Things the save cannot know** (player collected output in-game since the save): the
  projection is wrong until the next save, then snaps back. This is accepted (SPEC-0013).

## Verification

1. Note a running refiner's `projected_output` and `remaining_seconds` in `/json`; wait 60 s;
   `remaining_seconds` drops by ~60 and `projected_output` rises by `floor(60/spo) * per_op`.
2. In game, open that refiner after the projected finish time: the output stack equals
   `projected_output`. Survival mode uses slower rates (`seconds_per_op_survival`); check the mode.
3. Close the game; `/json` farm `fill_ratio` keeps rising and stops at exactly 1.0.
4. Load the game, save: values jump to the saved truth; for a measured farm the jump should be
   small (a few percent).
5. A plant planted at a known time is `mature` exactly `grow_seconds` later.

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md). The top one here: a farm's
assumed class-C rate (250 items/h per extractor) overestimates networks of 10+ extractors by 2-3x (13 gas extractors: 3,250/h assumed, 1,270/h
measured), so
the projection must switch to a measured rate as soon as two saves allow it. Also respect the
refiner output cap (9,999 for substances): the game stalls a refiner with inputs left when its
output slot is full, and a projection without the cap would show it running.
