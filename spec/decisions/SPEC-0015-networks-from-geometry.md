---
id: SPEC-0015
title: 'Extraction networks come from pipe geometry alone, not from fill levels'
description: 'Farm rows and the farm check trace pipe networks by segment-end distance; differing fill levels are never read as disconnection.'
type: decision
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0005, SPEC-0019]
---

# SPEC-0015: Extraction networks come from pipe geometry alone

## Context

The save has no notion of a resource network. It has extractors, depots and pipe segments, each
with a position (pipes also an `At` vector to their far end), and a fill level in `UserData`.
Two signals are available for "which units share a network": geometry (segment ends near units
and other ends), and fill levels. An early version observed that extractors on one network
carried the same fill on every base checked, and added a pass that split a traced component
wherever extractor fills disagreed, attaching depots to the nearest fill group. The farm check
likewise flagged a piped extractor whose fill was unique on its network as "split off".

Then two saves of one base 204 s apart were compared: three traced networks, and every unit in
each rose by exactly the same amount while keeping different absolute fills. Units on one
network gain in lockstep but keep the offsets they started with (built at different times,
emptied separately). The fill split had invented about a dozen one-extractor networks on that
base, each with a wrong rate and a wrong time-to-full.

## Decision

Geometry is the whole answer. Pipe ends within 3.2 units (the game joins within 3) of a unit or
another pipe end are connected; components are networks. Fills are summed per network but never
used to split or join. The "split off" check is removed from the farm check.

Units that merely touch are not joined (doing so lumped neighbouring networks' depot rows
together). Depot clusters on no extractor's network go to the nearest network by centre.

## Alternatives considered

- **Split by fill** (the previous behaviour): wrong as shown above.
- **Join on co-movement across saves** (units whose fills rise by the same amount are one
  network): correct in principle and a good *check*, but needs two saves with running extractors
  and fails for full or idle networks.
- **Join touching units**: merges distinct networks.

## Consequences

- Network rows match what the player sees in game; measured rates are per real network.
- A pipe that visibly stops short in the save (gap > 3.2) splits a network; the farm check
  reports the gap in metres so it can be fixed or the tolerance revisited.
- Revisit if the game's connection rule turns out to involve snap points the save does not
  record (wires already do: saved wires stop 4-15 m short of generators, which is why power is
  not traced).
