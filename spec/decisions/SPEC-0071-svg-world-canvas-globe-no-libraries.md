---
id: SPEC-0071
title: 'SVG world in metres plus a canvas globe, with no mapping or charting libraries'
description: 'Why the flat views are one SVG group in map metres with screen-pixel marks rescaled on zoom, the terrain and globe are canvases, and no Leaflet, d3, three.js or CDN is used.'
type: decision
area: map-ui
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0060, SPEC-0062, SPEC-0064, SPEC-0065, SPEC-0070, SPEC-0012]
---

# SPEC-0071: SVG world in metres plus a canvas globe, with no mapping or charting libraries

## Context

The map draws a few hundred to a few thousand objects per view: base parts, footprints, finds,
a trail of up to 4000 points, sector edges, a terrain raster. It needs pan, zoom, pinch,
per-mark hit targets, rotation of the whole world about the player (facing-up), labels that stay
upright and declutter, and a globe. The coordinate systems are custom (azimuthal equidistant
about the bases; base-local metres) rather than web-mercator tiles. The exporter's project rule
is standard library only on the server, and the page must load with no network access beyond the
exporter.

## Decision

- **Flat views are one SVG.** A `#world` group holds every layer in **map metres** (east right,
  north up, SVG y = −north). Pan and zoom only change the `viewBox`. Facing-up is one `rotate()`
  on `#world`, with labels counter-rotated about their anchors.
- **Marks are sized in screen pixels**: each element records its pixel radius, stroke width, dash,
  font size or icon size in `data-*` attributes, and a `rescale()` pass multiplies them by
  metres-per-pixel after every zoom.
- **Hit testing is the browser's**: invisible circles in hit layers above the marks.
- **Rasters are canvas, placed in the SVG**: the terrain surface is computed on a canvas
  (interpolation, hillshade), upsampled, and embedded as one `<image>` in map units; contours are
  SVG paths.
- **The globe is a separate canvas** with a hand-written orthographic projection, graticule,
  Rodrigues rotation for dragging, and its own hit list.
- **No libraries**: no Leaflet/MapLibre, no d3/d3-geo, no three.js, nothing from a CDN. Vector
  maths is ~20 lines (normalise, dot, cross, rotate).

## Alternatives considered

- **Leaflet with a simple CRS.** Good pan/zoom and markers, but rotating the whole map is not
  native, the custom projections fight its tile model, and it adds an external dependency or a
  vendored copy.
- **d3 / d3-geo.** Would give orthographic projection and zoom behaviours, but the page uses a
  small fraction, and the azimuthal equidistant centre is already computed server-side.
- **Everything on canvas.** Faster for very large counts, but loses free hit testing, text
  rendering, CSS styling and devtools inspection; the counts here don't need it.
- **WebGL/three.js globe.** Far more than a lit disc with lines and icons needs.

## Consequences

- Zero dependencies, works offline, nothing to update, and every behaviour is in one readable
  file.
- `rescale()` touches every mark on every zoom and wheel tick; fine at current counts (low
  thousands), would need batching beyond ~20k elements.
- Label decluttering uses `getBoundingClientRect()` per label after each redraw; same scaling
  caveat.
- Hand-written pointer handling had to learn the pointer-capture and drag-direction lessons
  that a library would have encoded (SPEC-0079).
- Revisit if the trail or terrain grows to tens of thousands of drawn elements, or if
  tile-based imagery of the planet ever becomes available.
