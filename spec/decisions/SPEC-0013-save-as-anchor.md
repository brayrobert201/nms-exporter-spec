---
id: SPEC-0013
title: 'The save is an anchor, not a feed: project forward from its timestamps'
description: 'Time-dependent numbers are computed per request from the save''s own per-object timestamps and the game''s rates, rather than shown as last saved or polled from the running game.'
type: decision
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0003, SPEC-0005, SPEC-0009]
---

# SPEC-0013: The save is an anchor, not a feed

## Context

The game writes a save every few minutes while playing (autosave on certain actions, manual saves)
and never while closed. The things the player wants to watch on a wall screen are timers:
refiners finishing, extraction farms filling, crops maturing, frigates returning. Shown as last
saved, every one of them is frozen and wrong by an unknown amount; the screens would only be
right at the moment of a save. Reading the running game's memory would be live but only while
it runs, and is fragile (that path exists in the `live-hook` area for position, not for this).

The save carries what is needed to do better: each refiner buffer has `LastUpdateTimestamp`, each
base object a `Timestamp`, each expedition a `StartTime`; the game tables give the rates
(`TimeToMake`, extractor 100 units/s, grow seconds, 5,400 s per expedition event).

## Decision

Treat the save as an **anchor**. Build a static snapshot once per save; on every request project
each time-dependent row from its own anchor to "now" at the game's rate, capped at its capacity
(SPEC-0003). Keep the saved value and anchor next to every projected one. Show the save's age on
every surface. When a new save arrives, everything snaps back to the truth.

Where the rate is not knowable from the save (extraction depends on hotspot class, which is not
saved), measure it from consecutive saves and fall back to a labelled assumption until then.

## Alternatives considered

- **Show last-saved values.** Simple and never wrong about what was saved, but useless for a
  timer screen; it was the starting point and the reason for this decision.
- **Poll the running game.** Live, but only while it runs; adds an in-process hook as a
  dependency of the screens. Kept for position/weather only.
- **Interpolate between the last two saves.** Needs two saves before showing anything and still
  needs a rate model for extrapolation; the measured-rate path does this where it matters (farms).

## Consequences

- Screens keep moving between autosaves and while the game is closed; refiners, farms and plants
  reach their caps on time.
- Anything the player did after the save (emptied a refiner) is wrong until the next save. This
  is accepted and made visible by the save-age line.
- Rates must be right: refiner times were checked against the game; farm rates are measured
  because the assumed class-C rate overstated big networks 2-3x.
- Event outcomes that the game only resolves while running (expedition events) are shown as
  saved, never projected.
- Revisit if the game starts writing saves continuously or if a live source becomes reliable.
