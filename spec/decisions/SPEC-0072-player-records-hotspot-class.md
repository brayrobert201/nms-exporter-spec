---
id: SPEC-0072
title: 'Hotspot class and resource are recorded by the player, never guessed'
description: 'Why the map shows hotspot class and mineral resource as unknown until the player records them (from the map or an in-game chat command), and how the two sources are merged.'
type: decision
area: map-ui
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0063, SPEC-0066, SPEC-0045]
---

# SPEC-0072: Hotspot class and resource are recorded by the player, never guessed

## Context

The point of the finds layer is choosing where to build extractors and generators, and that
depends on two facts per hotspot: its class (C/B/A/S sets output: e.g. 150–300 kP for power) and,
for minerals, which of the planet's three resources it gives.

- The hotspot marker the game adds carries type and position but **no class**: a class B power
  marker and a class S gas marker logged as raw structs differ only in position, label and type.
- The first version guessed a mineral hotspot gives the planet's common resource. On another
  planet a class B mineral hotspot gave the **rare** one. The guess was wrong half the time it
  mattered.
- The player sees both facts in the analysis visor in game.

## Decision

- The map shows class as unknown (`?`, "Class: set it from the visor") and a mineral hotspot's
  resource as the list of possibilities ("Copper or Ammonia or Silver"), with buttons to record
  the truth. Gas hotspots list the biome's two gases; power and deposits are certain.
- Two ways to record: buttons in a hotspot's details on the map (`POST /api/finds/class`, stored
  in the history store), and an in-game chat command `/hs S silver` (logged by the hook with the
  player's position, applied to the nearest hotspot within 50 m).
- Chat labels are kept as log events, never copied into the stored labels. Per field, the newer
  of the two wins, so a correction on the map is not undone by replaying an older chat line, and
  a later chat line overrides an older map choice.
- Raw marker structs are kept for labelled hotspots, as examples for anyone who later finds the
  class byte.

## Alternatives considered

- **Guess from the planet** (common resource, class C). Wrong in practice; a confident wrong
  answer sends the player to build in the wrong place.
- **Keep reverse-engineering the marker** until the class appears. Still open; the decision
  doesn't block it, and labelled examples help it.
- **Only map-side recording.** Needs the phone in hand at the hotspot; the chat command works
  with the controller.

## Consequences

- Unrecorded hotspots look unfinished (`?`), which is honest and nudges recording.
- Two sources need a merge rule (newest per field) and timestamps on map labels.
- A chat label far from any hotspot is reported as unmatched rather than dropped silently.
- Revisit when the class is found in game memory or files; then recorded labels become a check
  on the decoded value.
