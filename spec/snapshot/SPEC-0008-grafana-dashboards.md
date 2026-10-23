---
id: SPEC-0008
title: 'Grafana dashboards: status and crafting'
description: 'Two generated Grafana dashboards over the nms_* metrics: a portrait-first status board and an item-driven crafting board, both on the validated dark palette.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0002]
supersedes: []
superseded_by: []
related_specs: [SPEC-0006, SPEC-0009, SPEC-0019]
screenshots: []
---

# SPEC-0008: Grafana dashboards: status and crafting

## Summary

The exporter's metrics are scraped every 60 s into a Prometheus-compatible time-series database
and read through Grafana. Two dashboards are **generated** by Python scripts into JSON (edit the
generator, never the JSON), so layout and colours stay consistent and can be rebuilt after a
metric change. The status dashboard is portrait-first so it reads on a phone; the crafting
dashboard answers "can I make N of this, and what am I missing".

## Inputs and outputs

- Input: the `nms_*` families of SPEC-0002 in a datasource of type `prometheus` with uid
  `prometheus`.
- Scrape job: `scrape_interval: 60s`, one static target `<gaming PC>:9877`.
- Generators: `scripts/nms-grafana-dashboard.py [OUT.json]` -> `grafana/no-mans-sky.json`
  (uid `no-mans-sky`), `scripts/nms-grafana-crafting-dashboard.py` ->
  `grafana/no-mans-sky-crafting.json` (uid `no-mans-sky-crafting`). Standard library only.
- Delivery: import by hand, or provision from wherever Grafana reads dashboards (a ConfigMap for
  a sidecar, a provisioning directory).

## Requirements

### SPEC-0008-REQ-001: Generated, deterministic JSON

Each generator shall build the dashboard from small helpers (`row`, `stat`, multi-value `stats`,
`table`, `bargauge`, `timeseries`), assign panel ids sequentially, and write pretty JSON. Default
time range `now-3h`, refresh `auto`. The two dashboards link to each other.

### SPEC-0008-REQ-002: Palette

Colours shall come from the validated dark palette used by the wall screens (SPEC-0009): series
`#3987e5 #d95926 #199e70 #c98500 #d55181 #008300 #9085e9 #e66767` in that fixed order, ink
`#c3c2b7` for values, and status hues `good #0ca30c`, `warning #fab219`, `serious #ec835a`,
`critical #d03b3b` used **only** for state (thresholds on things like save age or collector up),
never to distinguish series. Magnitude bars stay series-1 blue. One axis per chart; direct labels
where they fit.

### SPEC-0008-REQ-003: Status dashboard rows

In order, each a collapsible row:

| Row | Panels |
|---|---|
| Right now | Where (save summary), Save age, Portal glyphs, Galactic address, Currency (multi-value), Time, Vitals bar gauge, Ship |
| Refiners (projected from the save's timestamps) | counts; table (recipe, base, output, input left, remaining, finish time); timeseries of input left |
| Extraction farms (projected) | storage-used bar gauge; counts; table with rate and measured/assumed; fill over time |
| Fleet | counts; voyage progress; expeditions table; frigates table with base-roll score |
| Inventory | counts; top 20 bar chart; "Where is it?" table for `$item`; totals; totals over time |
| Plants | counts; plants by kind with yield and stock |
| Missions | counts by status; in progress, closest to done first |
| Technology | placement by inventory; moves worth making; families not touching |
| Tidy storage | counts; containers; moves |
| Almost craftable | known blueprints short of stock, by the ingredient in the way |
| Lifetime | discovery, combat, distance, earned ever; units, nanites, collections over time |
| Exporter | status (`nms_collector_up`), decoder (decodes, failures, duration), save age over time |

Multi-value stat panels wrap to fit, so several values share one panel on a phone. Templating:
`item` (from `nms_item_total`) and `type`.

### SPEC-0008-REQ-004: Crafting dashboard

Variables: `item` (query `nms_craftable`), `qty` (textbox, default 1), `known`
(`known : 1, all : .*`), `route` (craft / refine / cook filter). Rows: Overview (counts; "What can
I make now?" table of `nms_craftable > 0`), Selected item (ingredients per route for `$qty`, raw
materials with every intermediate expanded, shortfall for one unit, what uses the item, can-make
over time), Everything (every recipe).

Per-route ingredient need for `$qty` is the join

```
nms_recipe_need{id="$item"} * on (id, route, recipe_id) group_left()
  ceil($qty / nms_recipe_yield{id="$item"})
```

and "have" joins on `ingredient_id` against `nms_item_total`.

### SPEC-0008-REQ-005: Select by id, display by name

Item variables shall carry the item **id** as value and the name as text, extracted with a regex
that does not depend on label order:

```
/id="(?<value>[^"]+)".*?item="(?<text>.*?)"(?:,|})/
```

Queries filter on `id="$item"`; titles use `${item:text}`. Joins use `recipe_id`, never the
display `recipe` name (several recipes share a display name).

### SPEC-0008-REQ-006: Honest blanks

Panels over projected values shall say so in their title or description. Count panels use
`... or vector(0)` so "none" shows as 0 rather than "No data"; tables with nothing missing say
"Nothing missing" instead of an empty grid.

## Degraded behaviour

- **Exporter down**: `nms_collector_up` absent/0 and the Exporter row shows it; every other panel
  shows its last scraped values until they age out of the time range. Save age keeps rising in
  the exporter's own reading but not in Grafana, which is how to tell "exporter down" from
  "game closed".
- **Save unreadable**: `nms_collector_up 0`, all game panels "No data" (SPEC-0002-REQ-010).
- **Crafting still computing**: the crafting dashboard is blank for a few seconds after a save.
- **Datasource uid differs**: panels show "datasource not found"; change `DS` in the generators.

## Verification

1. Regenerate both JSON files and `git diff` them: only intended changes.
2. Import into a scratch Grafana with a Prometheus datasource (uid `prometheus`) scraping the
   exporter; every panel renders with no "No data" while the game has a valid save.
3. Pick an item whose name contains a quote or apostrophe in the crafting dashboard; the selected
   row still populates.
4. Open the status dashboard on a phone-width window: multi-value panels wrap, no horizontal
   scroll.

No screenshot is included: rendering needs a running Grafana and time-series store, which this
spec does not assume.

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): quoted item names broke label
matchers (select by id); the reused recipe display name broke joins (join on `recipe_id`); the
variable regex assumed label order.
