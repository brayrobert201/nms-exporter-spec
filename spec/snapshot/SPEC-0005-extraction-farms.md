---
id: SPEC-0005
title: 'Extraction farms: pipe networks, fill, rate, and the farm check'
description: 'Group extractors and supply depots into pipe networks from geometry, read fill levels from UserData, measure rates from consecutive saves, name rows from the base name, and flag loose extractors.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0001, SPEC-0002]
supersedes: []
superseded_by: []
related_specs: [SPEC-0003, SPEC-0015, SPEC-0019]
screenshots: [conky-bases.png]
---

# SPEC-0005: Extraction farms: pipe networks, fill, rate, and the farm check

## Summary

A "farm" is a set of mineral or gas extractors on a survey hotspot, joined by pipelines to supply
depots that buffer the output. The player wants three numbers per farm: how full it is, how fast
it fills, and when it will be full. The save stores every base part with a position and a packed
`UserData`, and nothing else: not which hotspot an extractor sits on, not its class, not which
substance it yields, not which network it belongs to. The exporter rebuilds networks from pipe
geometry, sums fills, measures the rate from consecutive saves, and names the row from the
substance words the player put in the base name. A companion check lists extractors that are not
properly hooked up.

## Inputs and outputs

Per base in `PersistentPlayerBases[]`: `Name`, `LastUpdateTimestamp`, `Objects[]` where each
object has `ObjectID`, `Position [x,y,z]` (base-local), `At` (for pipes), `UserData` (int64),
`Timestamp`.

| ObjectID | Role | Capacity (resource units) |
|---|---|---|
| `^U_EXTRACTOR_S` | mineral extractor | 360,000 (250 items) |
| `^U_GASEXTRACTOR` | gas extractor | 360,000 (250 items) |
| `^U_SILO_S` | supply depot | 1,440,000 (1,000 items) |
| `^U_PIPELINE` | pipe segment: one end at `Position`, the other at `Position + At` (`At` is not normalised; its length is the segment length) | - |
| `^U_POWERLINE` | wire segment, same geometry | - |
| `^BASE_FLAG` | base computer (origin for the farm check) | - |

Constants (from the tables, with these defaults): `resource_units_per_item = 1440`, extractor
`rate_per_s = 100` units (250 items/h at class C), extractor and depot storage as above. Hotspot
class multiplies the rate by 1 / 1.5 / 2 / 2.5 (C/B/A/S) but is **not in the save**.

**Fill** of an extractor or depot = `UserData >> 32` (upper 32 bits), in resource units.

Output (`/json farms`, keyed by farm key): `base`, `name`, `kind` (`mineral`/`gas`),
`extractors`, `gas_extractors`, `depots`, `stored_units`, `capacity_units`, `stored_items`,
`capacity_items`, `anchor`, `rate_units_per_s`, `rate_measured`, `rate_items_per_hour`, plus the
projection (SPEC-0003).

## Requirements

### SPEC-0005-REQ-001: Networks from pipe geometry only

For each base with extractors or depots, the exporter shall build connected components with
union-find: each pipe end within **3.2** units (the game joins within 3) of a unit's `Position`
joins that unit; each pipe end within 3.2 of another pipe's end joins the two pipes. Units that
merely touch are **not** joined. Fill levels shall **not** be used as a connection signal
(SPEC-0015).

### SPEC-0005-REQ-002: Groups from components

- Components containing at least one extractor become groups. An unpiped extractor is its own
  component and so its own (small) group.
- Each unit of a component with no extractor (a depot field nobody pipes to) shall join the
  extractor component whose centre (mean position) is nearest.
- Group kind = `gas` if gas extractors >= mineral extractors, else `mineral`.
- If the base has **no pipes at all**: with both mineral and gas extractors, make two groups by
  type and give each depot to the type whose extractor centre is nearer; otherwise one group.
- Order groups by extractor count, descending.

### SPEC-0005-REQ-003: Row naming

The save does not record what a hotspot yields, so names come from the base name:

1. Normalise the base name to lowercase words (`[^a-z0-9]+` -> space).
2. Match every substance name from the tables, **longest first**, as a whole-word phrase,
   removing each match so "Magnetised Ferrite" is not also read as "Ferrite".
3. Substances `GAS1`-`GAS4` and `OXYGEN` (what a gas extractor can yield) go to the gas list,
   all others to the mineral list.
4. Row name = the words for the group's kind joined by spaces; if none, `"<base> (<kind>)"` when
   the base has more than one group, else the base name.
5. The n-th group of the same kind in a base gets suffix ` n` (n >= 2): "Oxygen", "Oxygen 2".
6. Farm key = `"<base> · <kind>"` when the base has more than one group, else `<base>`, plus the
   same suffix. Keys must be stable across saves: they index the rate history.

### SPEC-0005-REQ-004: Sums and anchor

Per group: count each type, `stored_units = Σ fill`, `capacity_units = Σ capacity`,
`anchor = max(base.LastUpdateTimestamp, every member's Timestamp)`.

### SPEC-0005-REQ-005: Measured rate

Per farm key the state file keeps `[[anchor, stored_units], ...]`, appending when the anchor
changed and keeping the last **60**. Over the last 30 consecutive pairs, a pair with
`t1 > t0 and u1 > u0 and u1 < capacity_units` gives `rate = (u1 - u0) / (t1 - t0)`; the latest
such pair wins and `rate_measured = true`. Otherwise
`rate = 100 * (extractors + gas_extractors)` units/s, `rate_measured = false`.
`rate_items_per_hour = rate * 3600 / 1440`.

The rate is re-measured once per new save (the anchor changes), not on a timer.

### SPEC-0005-REQ-006: Farm check

A shared function shall report, for every extractor in a base, a list of problems:

| Problem | Test |
|---|---|
| `not piped` | no pipe end within 3.2 of it (it fills its own 250 items and stops) |
| `no depot` | piped, but its component has no `^U_SILO_S` |
| `no wire` | no `^U_POWERLINE` end within **2.5** of it (wire ends sit 0.2-2.2 from the part they feed) |
| `never filled` | fill is 0 (unpowered, off the hotspot, or just built) |

Each row also carries `kind`, `network_size`, `fill_items`, position relative to the base
computer (`x`, `z`, `from_computer_m = hypot(x, z)`, `height_m = y`), `pipe_gap_m` and
`wire_gap_m` (distance to the nearest pipe / wire end, so a small gap means "stops just short"),
and for problem rows `nearest_network` = `(distance, kind, size)` of the nearest other network
with more than one extractor. Rows sort problems first, then kind, then distance.

The check shall not trace power to a generator: on working farms the saved wires stop 4-15 m
short of the generators (power passes through parts or snap points the save does not describe).
An extractor at 250/250 is normal: its own buffer stays full while the depots fill.

### SPEC-0005-REQ-007: Farm check CLI

`nms-farm-check [text] [--all]` shall read `/raw/bases` from the exporter (`NMS_EXPORTER_URL`),
filter bases by name substring, and print problem rows (all rows with `--all`). The map page
(`map-ui` area) uses the same function to flag loose extractors.

## Degraded behaviour

- **No pipes in a base**: grouped by type and proximity (REQ-002), still one or two sensible rows.
- **No substance word in the base name**: the row is named after the base; nothing breaks.
- **Base renamed or networks re-numbered**: the farm key changes, history restarts, the rate
  is assumed again until two more saves.
- **State not writable**: every rate stays assumed (`rate_measured: false`); screens say
  "assumed C-class" so the number is not mistaken for a measurement.
- **Extractors full between saves**: a pair whose later fill is at capacity is ignored, so a full
  farm never yields a falsely low rate.

## Verification

1. For a farm you can visit: sum the depot readings in game; `stored_items` in `/json` matches
   at the save time (`nms_farm_stored_items_saved`).
2. Two saves a few minutes apart on a running farm: every unit on one network rises by the same
   amount; `rate_measured` flips to true and `rate_items_per_hour` is near the in-game estimate.
3. Disconnect one pipe in game, save: `nms-farm-check <base>` lists the extractor as
   `not piped` with a `pipe_gap_m` of a few metres and the network it should join.
4. Network count per base matches what you see in game (Cobalt Oxygen: three traced networks).

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): splitting networks by differing
fills invented a dozen one-extractor networks; the assumed class-C rate overstates big networks
2-3x; the save holds no hotspot class or yield, so names come from the base name.
