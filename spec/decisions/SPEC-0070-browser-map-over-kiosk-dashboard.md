---
id: SPEC-0070
title: 'An interactive browser page served by the exporter, not a kiosk dashboard'
description: 'Why the map is a self-contained HTML page served by the exporter itself, used on a phone, instead of a panel on the fixed wall screens or in the metrics dashboard tool.'
type: decision
area: map-ui
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0060, SPEC-0061, SPEC-0069, SPEC-0071, SPEC-0008, SPEC-0009, SPEC-0085]
---

# SPEC-0070: An interactive browser page served by the exporter, not a kiosk dashboard

## Context

The exporter already fed two kinds of display (save-exporter area): a metrics dashboard tool
reading time series, and fixed portrait wall screens rendered in kiosk mode. Both are glanceable
and hands-off: no input, fixed size, refreshed every few seconds.

The map's job is different:

- It is used **while playing**, usually on a phone held next to the game, to find the way back to
  a base, a hotspot, or the next uncharted sector.
- It needs **input**: pan, zoom, pinch, tap a mark for details, set a waypoint, record a
  hotspot's class, switch planets.
- It needs **live position** at a few hertz with a facing-relative arrow, which neither a
  time-series dashboard nor a 5-second kiosk refresh can give.
- Its geometry is a sphere and base-local frames, not time series.
- It must be reachable from any device on the network without installing anything.

## Decision

Serve `/map` and `/systems` as single self-contained HTML files (inline CSS and script, no build
step, no external requests) **from the exporter's own HTTP server**, alongside JSON APIs shaped for
them (`/api/map`, `/api/live`, …). The page is read from disk on each request so it can be edited
without a restart. It borrows the wall screens' palette tokens and design rules (SPEC-0069) so the
products look like one system. A browser hitting `/` with `Accept: text/html` is redirected to
`/map`.

## Alternatives considered

- **A panel in the metrics dashboard tool** (geomap or canvas panel). No custom projections
  (azimuthal equidistant about bases, base-local north-up), weak touch interaction, no
  write-back for hotspot classes, and polling granularity tied to the tool's refresh.
- **A third wall screen.** Fixed in place and size; the player needs it in hand while walking.
- **A separate web app with a framework and bundler.** More moving parts than the problem needs,
  a build step that must be kept working, and another service to deploy; the exporter already has
  every input.
- **A native or in-game overlay.** Platform-specific, and an overlay inside the game under a
  compatibility layer is fragile (live-hook area).

## Consequences

- One process serves save metrics, history and the UI; the map works wherever the exporter does.
- The page is one large file (~1700 lines). It stays manageable because each layer is a section
  with its own load/draw functions; split it if a second contributor appears.
- No offline caching or install; acceptable on a home network.
- The exporter's HTTP server is now user-facing: it needs to stay responsive (threaded server,
  cached computations) and must answer JSON errors, not stack traces.
- Exposing it beyond the home network needs a reverse proxy with authentication (storage-deploy
  area); the page itself has none.
- Revisit if the map needs offline use away from the network, or if several people use it at
  once with different saves.
