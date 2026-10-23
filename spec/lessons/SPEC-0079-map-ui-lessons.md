---
id: SPEC-0079
title: 'Map UI lessons'
description: 'Gotchas from building the /map and /systems pages: query encoding, pointer capture, hit layer order, kind collisions, planet matching, globe drag direction, label clutter, live-layer signs and refresh behaviour.'
type: lesson
area: map-ui
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
related_specs: [SPEC-0060, SPEC-0061, SPEC-0062, SPEC-0063, SPEC-0064, SPEC-0065, SPEC-0066, SPEC-0067, SPEC-0068, SPEC-0069, SPEC-0043, SPEC-0049]
---

# SPEC-0079: Map UI lessons

Ordered by how much time each would save a second builder.

## Lesson: the ship's `at` points the other way

**What happened.** The live arrow and every Nearby turn arrow were correct on foot and reversed
while flying. Over the recorded trail, movement against the player transform's `at` averaged
cos −0.94 on foot (304 steps) and +0.99 in the ship (1042 steps, locations `PlanetInShip` and
`Default`).

**The rule.** Facing is `-at` on foot and `+at` in ship locations (`PlanetInShip`, `Default`,
`SpaceInShip`, `PlanetInAtmoShip`). Check any heading against measured movement, per location,
before trusting it.

## Lesson: the corvette and freighter are bases

**What happened.** The save lists the freighter and the player's corvette as bases, positioned
wherever they are now. A base-less planet with the corvette parked on it was counted as having a
base, so it was left out of the planet list and the live layer had no map to draw on.

**The rule.** Only planetary bases count: skip `FreighterBase`, `PlayerShipBase`,
`PlayerSpaceBase` everywhere a base defines a planet, a projection centre or a "bases in this
system" list.

## Lesson: pointer capture on press steals taps

**What happened.** The pan handler called `setPointerCapture` on `pointerdown`. From then on the
`click` went to the SVG, not to the mark under the finger, so tapping marks on a phone did
nothing.

**The rule.** Record the pointer on press; capture only once it has moved more than 4 px. While
following, ignore movement under 8 px so a tap doesn't cancel Centre.

## Lesson: the finds hit layer must sit above built objects' targets

**What happened.** Extractors are built on top of their hotspot, so the extractor's 28 px hit
circle covered the hotspot's. Tapping a hotspot opened the extractor.

**The rule.** Separate hit layers per family, finds above built objects. Whenever two kinds are
co-located by design, decide which wins the tap and order the layers for it.

## Lesson: two different things called "deposit"

**What happened.** The save's drained deposits (kind `deposit`, "Drained deposit") and visor
`TerrainResource` finds (also `deposit`) shared one entry in the kind table: same colour, same
legend chip, and hiding one hid the other.

**The rule.** Give each source its own client kind (`deposit-find` for the visor's) and map at
the boundary. Grep for kind collisions whenever a new layer arrives.

## Lesson: the globe's drag direction took two tries

**What happened.** The first globe turned correctly horizontally but rolled up when swiping down.
The first "fix" reversed the **horizontal** direction, which had been right, so both axes were
then wrong in different ways. The fix that stuck reverted that and flipped the vertical:
`gc = rotate(rotate(gc, north, -dx/R), east, -dy/R)`.

**The rule.** Specify drag as "grab": the surface point under the finger stays under the finger.
Test both axes by putting a finger on a mark and dragging it to each edge before changing signs.

## Lesson: space vs plus in query strings

**What happened.** The page builds queries with `URLSearchParams`, which encodes a space as `+`.
Base names have spaces. Python's `urllib.parse.unquote` leaves `+` alone, so `base=Silver+Ridge`
names no base and the layout, target and finds come back empty with no error.

**The rule.** Decode query values with `unquote_plus` (a literal `+` arrives as `%2B`). Test
every endpoint with a name containing a space.

## Lesson: `?planet=` must match more than base names

**What happened.** `?planet=` matched only base names, so a base-less planet could not be linked
to at all. Its label is generated (`Okyota Gamma in Ubunji (no bases)`).

**The rule.** Match base names, `planet_name`, the exact label, and a label prefix followed by a
space.

## Lesson: labels pile up; declutter in screen space by priority

**What happened.** With hotspots labelled `A Silver`, `B 220 kP`, sector names and base names,
zoomed-out views were unreadable, and in facing-up mode map-space overlap tests were wrong.

**The rule.** Give every label a priority (marks/problems 5, bases 4, hotspots 3, default 2,
sectors 1), measure screen bounding boxes after each draw, hide losers; group finds within 26 px
under a count badge. Re-run on every zoom and rotation.

## Lesson: don't hide the Globe button in base view

**What happened.** The Globe button only existed in planet view; on a one-base planet the page
opens straight into the base layout, so the globe was effectively unreachable.

**The rule.** Keep navigation buttons present in every view; from a base, Globe goes to the
planet first.

## Lesson: nominal radius is the most common value, not the latest

**What happened.** `|rel| - distance_from_planet` is the planet's nominal radius, but ship and
underwater samples were off by up to 24 m and the latest sample won, shifting every height.

**The rule.** Take the mode of `round(|rel| - dfp)` (818 of 1002 samples agreed on one planet).

## Lesson: the lowest heights vanished against the page

**What happened.** A full sequential ramp's darkest step was indistinguishable from the
background, where "no colour" means "never sampled".

**The rule.** Use steps 600–200 of the sequential ramp only, and draw nothing where unsampled.

## Lesson: a 2 Hz redraw restarts CSS animations

**What happened.** The live layer is rebuilt every poll; a new pulse element starts its animation
at 0, so the pulse stuttered and never completed.

**The rule.** Phase the animation to the wall clock: `animation-delay: -(Date.now() % period)`.

## Lesson: the dock covers the content

**What happened.** On a phone the Nearby and power cards covered the lower half of the fitted
view; centring put the player under the cards.

**The rule.** Measure what the dock hides in the map's middle third and fit/centre into the
remaining area; refit an untouched view when the dock first appears or changes size.

## Lesson: the page must find planets that appear after it loaded

**What happened.** Landing on a new planet left the page saying "another planet" because its
planet list was from load time.

**The rule.** When the live address is not in the list, reload `/api/map` (at most once a minute
per address, so a planet the server can't place doesn't cause a reload storm).

## Lesson: a live list re-rendered under an open control closes it

**What happened.** The Places sheet refreshes with the 0.5 s poll; re-rendering replaced its
`<select>` while it was open.

**The rule.** Throttle list refresh (≥ 2 s) and skip it while focus is inside the list's
controls.

## Lesson: north had to be measured, and said so until it was

**What happened.** The first map assumed north = planet +Y and labelled the arrow `N?`. A live
walk at a base confirmed it (east bearing 89.4°).

**The rule.** Show unverified orientation as unverified; verify with a compass walk, then flip a
single flag.
