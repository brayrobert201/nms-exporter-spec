---
id: SPEC-0069
title: 'Map and systems pages: design rules, palette tokens and degraded states'
description: 'Dark-only design rules for /map and /systems: colours as CSS custom properties from a validated palette, status hues reserved for state and never colour-alone, ink tokens for values, neutral blue for magnitude, glass panels, phone and fold widths, and the database-down banner and 503 handling.'
type: snapshot
area: map-ui
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: []
supersedes: []
superseded_by: []
related_specs: [SPEC-0060, SPEC-0062, SPEC-0064, SPEC-0066, SPEC-0068, SPEC-0070, SPEC-0009, SPEC-0081]
screenshots: [map-phone.png, map-fold.png, power-check.png, storm-countdown.png, facing-up.png]
---

# SPEC-0069: Map and systems pages: design rules, palette tokens and degraded states

## Summary

Both pages follow one small design system so they read as one product, stay legible at night
next to a game, and never mislead: colour carries category or magnitude, status hues carry state
and nothing else, and every state is also said in words. The pages must work on a portrait phone
in one hand (≈ 400 px), on a folding phone's inner and outer screens, and on a desktop. When part
of the backend is down the pages say what is missing instead of drawing empty or zero data.

## Inputs and outputs

None beyond the pages. The palette below is a validated dark palette from a data-visualisation
design method (category hues checked for mutual distinguishability and contrast on the dark
surface). A rebuild may substitute another validated palette; the token names and rules are the
spec.

## Requirements

### SPEC-0069-REQ-001: Dark only

The pages shall be dark-only, with `--page` as the body background, and shall not offer a light
theme.

### SPEC-0069-REQ-002: Colours as tokens

Every colour shall be defined once as a CSS custom property on `:root` and referenced by name
(`var(--x)` in CSS, a `css("--x")` lookup in script). No hex value shall be introduced ad hoc in
markup or drawing code. Tokens:

| token | value | role |
|---|---|---|
| `--page` | `#0d0d0d` | background, label halos |
| `--surface-1` / `--surface-2` | `#1a1a19` / `#232321` | panels / controls |
| `--ink-1` / `--ink-2` / `--ink-muted` | `#ffffff` / `#c3c2b7` / `#898781` | text and values |
| `--grid` / `--axis` | `#2c2c2a` / `#383835` | grid lines / rules, outlines |
| `--border` | `rgba(255,255,255,0.10)` | control borders |
| `--series-1,2,3,4,7` | `#3987e5`, `#d95926`, `#199e70`, `#c98500`, `#9085e9` | categories |
| `--good` / `--warning` / `--serious` / `--critical` | `#0ca30c` / `#fab219` / `#ec835a` / `#d03b3b` | state only |
| `--seq-1…9` | `#184f95 #1c5cab #256abf #2a78d6 #3987e5 #5598e7 #6da7ec #86b6ef #9ec5f4` | sequential blue, steps 600–200 |
| `--div-below` / `--div-mid` / `--div-above` | `#3987e5` / `#383835` / `#e66767` | diverging around a reference |

### SPEC-0069-REQ-003: Status hues are reserved for state

`--good`, `--warning`, `--serious`, `--critical` shall be used only for state: live/stale, power
verdicts, loose extractors and other problems, storm phases, pirate/high conflict, dialable,
"you are here", database down. They shall never colour a category or a magnitude.

### SPEC-0069-REQ-004: Never colour alone

Every state shall be shown as a pill or label pairing the hue with a **glyph and a word**:
`● Live`, `⚠ Stale`, `○ No live data`, `✓ Covered day and night`, `⚠ Nights depend on biofuel`,
`✕ Short 120 kP at night`, `⚠ loose`, `⛈ Storm coming`, `☠ Pirate`, `⚠ High`, `✓ dialable`,
`⚠ Database down`. Pills: rounded, `--surface-2` fill, 1 px border and text in the hue.

### SPEC-0069-REQ-005: Values wear ink

Numbers, names and readouts shall use ink tokens (`--ink-1` for primary values, `--ink-2` for
secondary, `--ink-muted` for notes), never a series colour. Series colours are for marks
(dots, icons' fallbacks, footprints, segments).

### SPEC-0069-REQ-006: Magnitude is neutral blue

Magnitude (ground height) shall use the sequential blue ramp, or the diverging ramp when values
sit either side of a meaningful reference (the water surface), with the reference at the neutral
midpoint. Healthy or large values shall not turn green, nor low ones red.

### SPEC-0069-REQ-007: Truncated axes are labelled

Where a colour or axis scale does not start at a natural zero, the page shall drop any area fill
that implies zero and state the scale in words (SPEC-0062-REQ-010).

### SPEC-0069-REQ-008: Glass and depth

Floating panels (details, dock cards, away banner, map controls, tooltip) shall be
`color-mix(in srgb, var(--surface-1) 78%, transparent)` with a 10 px backdrop blur and a soft
`0 6px 24px rgba(0,0,0,0.35)` shadow; the Places sheet at 88 %. The map sits on a radial
vignette (SPEC-0060-REQ-013). Ambient tint from the planet never touches data marks.

### SPEC-0069-REQ-009: Widths

The pages shall work at ≈ 400 px portrait (phone), ≈ 900 px (fold inner screen) and desktop:
header and legend wrap; dock cards wrap upward and cap at 420 px; at ≤ 700 px dock text is 13 px,
the problems card max 22 vh, Nearby max 26 vh, and the power card starts collapsed; only the
systems table may scroll sideways, inside its own container. `viewport-fit=cover`; sticky headers
respect `env(safe-area-inset-top)`. The map uses `touch-action: none` so the browser does not
pan the page.

![Fold width](../../screenshots/map-fold.png)

### SPEC-0069-REQ-010: Database-down banner

When any `/api/*` response has `db: false`, the map shall show a banner below the away banner:
`⚠ Database down` (critical pill) `History, finds and sectors can't load: <reason>`, and hide it
again as soon as a response succeeds. `/api/map` reports the same through `errors["nms-db"]`
while still answering 200 with the save's planets. The systems page shows its own one-line
message instead of the table (SPEC-0068).

### SPEC-0069-REQ-011: 503 handling on the server

The history-backed endpoints (`/api/live`, `/api/elevation`, `/api/finds`, `/api/sectors`,
`/api/systems`) shall answer **503** with `{"error": "<reason>", "db": false}` when the history
store is not configured or any store error occurs while serving them; `POST /api/finds/class`
answers 503 `{"ok": false, "error": …}`. Save-backed output (`/api/map`'s save planets, the
metrics and JSON endpoints) shall never fail because of the store.

### SPEC-0069-REQ-012: Explain the blank

Every layer that can be empty shall say why in the legend or panel ("Heights: not known until you
visit this planet with the live_position mod running", "Nothing recorded on this planet yet.",
"○ No live data") rather than drawing zeros or nothing.

### SPEC-0069-REQ-013: Motion

Motion is limited to the live pulse and view glides; both respect `prefers-reduced-motion`
(pulse static; glide is short and never loops).

### SPEC-0069-REQ-014: Type

`system-ui` stack, 15 px body, 13 px secondary, tabular numerals for distances and numbers in
lists and tables.

## Degraded behaviour

This spec is largely about degraded behaviour; see REQ-010 to REQ-012.

## Verification

- `grep -E '#[0-9a-fA-F]{3,6}\b'` over the pages finds hex only inside the `:root` block.
- Turn the screen to greyscale: every state is still readable from its glyph and word.
- Stop the history store: within one poll the banner appears; `/metrics` still answers; the map
  still shows bases and layouts. Start it: the banner clears.
- Load both pages at 412 × 900: no horizontal page scroll; the map's header, dock and legend
  wrap; the systems table scrolls inside its frame.

![Phone width](../../screenshots/map-phone.png)

## Lessons

See [SPEC-0079](../lessons/SPEC-0079-map-ui-lessons.md).
