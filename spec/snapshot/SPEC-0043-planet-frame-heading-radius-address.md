---
id: SPEC-0043
title: 'Planet frame, compass, heading, nominal radius, sea level and universe address'
description: 'The measured facts that turn live_position samples into a map: rel is the save''s planet frame, north = planet +Y, east = north x up, the sign of at by location, nominal radius as the modal |rel| - distance_from_planet, sea level, and universe-address packing.'
type: snapshot
area: live-hook
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0042]
supersedes: []
superseded_by: []
related_specs: [SPEC-0045, SPEC-0046, SPEC-0049]
screenshots: []
---

# SPEC-0043: Planet frame, compass, heading, nominal radius, sea level and universe address

## Summary

The live samples (SPEC-0042) are raw world vectors. This spec fixes how they are interpreted, each
rule with the measurement behind it, so a rebuild can place the player on the same map as the save's
bases and markers, point the arrow the right way, and compute ground heights. Everything here was
measured in game on build 25800958 (NMS.exe 181442); where a rule rests on few observations it says so.

## Inputs and outputs

- In: `live.json` / trail samples (`rel`, `at`, `up`, `location`, `distance_from_planet`, `sealevel`,
  `ua`, `planet_pos`, `planet_index`, `in_atmosphere`).
- In: the save's `PersistentPlayerBases[].Position` / `GalacticAddress` (save-exporter area).
- Out: per sample - planet address, on-planet flag, heading (deg, 0 = north, clockwise), lat/lon,
  surface distance/bearing/turn to targets; per planet - nominal radius, water level.

## Requirements

### SPEC-0043-REQ-001: rel is the save's planet frame
The system shall treat `rel = mPlayerTM.pos - mNearestPlanetPos` as the same planet-centred frame the
save uses for a planetary base's `Position`.
Measured: player standing inside a base: rel (-103273.0, -2775.1, -79488.1), |rel| 130,351; the save
Position of that base (-103276.4, -2752.9, -79481.4), |P| 130,349; 21.6 apart. Freighter and corvette
bases in the save use a different frame (|Position| 1,642 and ~13.5k); exclude them.

### SPEC-0043-REQ-002: Compass north and east
At a point with unit up `u = rel/|rel|`, the system shall take
`north = normalise(Y - (Y.u) u)` with `Y = (0, 1, 0)` (fallback `(1, 0, 0)` at a pole) and
`east = north x u` (right-handed cross product, in that order). Latitude `= asin(u.y)`, longitude
`= atan2(u.x, u.z)` (increases eastward).
Measured with the in-game compass: facing north, the facing vector made tangent was
(0.005, 0.999, -0.042); facing east, the computed bearing was 89.4 deg. One planet (radius ~130.3k).

### SPEC-0043-REQ-003: Facing from mPlayerTM.at depends on location
The system shall take the player's facing as `-at` on foot and in every non-ship location, and as
`+at` when `location` is `PlanetInShip` or `Default` (flying, and the ship in space reports
`Default`). The facing is projected onto the tangent plane, normalised, and the heading is
`atan2(f.east, f.north)` in degrees, wrapped to [0, 360).
Measured over the trail: movement versus `at` averaged cos -0.94 on foot (304 steps) and +0.99 in the
ship (1,042 steps, locations `PlanetInShip` and `Default`); standing in a base, 15 of 18 walking steps
> 3 m were opposite `at` (cos -0.96 to -1.00). The corvette interior (`Freighter`) and `Underwater`
were mixed and keep the on-foot sign.
(`EnvironmentLocation` values in nmspy 181442.1: None_ 0, Default 1, SpaceStation 2, PlanetOnFoot 3,
PlanetInShip 4, PlanetInVehicle 5, Underwater 6, Cave 7, IndoorInBase 8, Freighter 9,
FreighterInternals 10, AbandonedFreighter 11, InFleet 12, InSpaceObject 13, Nexus 14, Anomaly 15.
The original exporter also lists `SpaceInShip` and `PlanetInAtmoShip`, which do not exist; harmless.)

### SPEC-0043-REQ-004: Corvette and freighter locations
The system shall label `location == "Freighter"` (9) as the player's corvette interior and
`FreighterInternals` (10) as the capital freighter. Observed: inside the corvette the game reports
`Freighter`.

### SPEC-0043-REQ-005: Nominal planet radius
For each planet address, for every sample with `distance_from_planet < 20000` and planet bits != 0,
the system shall count `round(|rel| - distance_from_planet)` and use the **most common** value as the
planet's nominal radius (not the latest). The values are multiples of 1024: 130048 = 127 x 1024,
98304 = 96 x 1024 seen; save bases cluster near ~64.6k, ~98.4k and ~130.1k.
Measured: 818 of 1,002 samples on one planet gave 98304; ship and underwater samples gave
98280-98303, so "latest wins" was off by up to 24 m.

### SPEC-0043-REQ-006: Sea level and ground height
The system shall take `sealevel` as the water surface height above the nominal radius: a planet with
`sealevel != 0` has a sea at radius + sealevel (120 seen on a water planet); 0 on every dry planet
seen, so 0 means no sea. Ground height of a sample = `|rel| - radius - (sealevel or 0)`. Only samples
with location `PlanetOnFoot` or `PlanetInVehicle` count as ground; per map cell keep the lowest
sample (jetpack jumps do not raise ground).

### SPEC-0043-REQ-007: On a planet or not
The system shall treat a sample as on a planet only when the universe address is known, its planet
bits are non-zero, and (`in_atmosphere` or `distance_from_planet <= 20000`). Planet bits 0 means "in
the system, at no planet" (space, galaxy map) whatever the distance says; never treat planet 0 as a
planet (no radius, no map, no name).

### SPEC-0043-REQ-008: Universe address packing
The system shall pack the mod's `ua` dict exactly as the save packs `GalacticAddress` (nmspy
`cGcUniverseAddressData.to_UA`):

```python
ua = (x & 0xFFF) | ((z & 0xFFF) << 12) | ((y & 0xFF) << 24) \
     | (reality << 32) | (system << 40) | (planet << 52)
# reality 8 bits, system 12 bits, planet 4 bits; x, z signed 12-bit, y signed 8-bit (two's complement)
```

and unpack with sign extension (`v - 2**bits if v >= 2**(bits-1)`). A system's address is the planet
address with bits 52-55 cleared. When `ua` is null (mid-game attach), the system shall look up the
rounded `planet_pos` in a map of planet centre -> address built from earlier samples that had both
(planet bits != 0), and mark the result as inferred.

### SPEC-0043-REQ-009: miNearestPlanetIndex
The system shall not use `planet_index` (`miNearestPlanetIndex`) as the address planet number. It
appears to be 0-based (0 on the planet whose address planet is 1; 2 on a planet whose name seed is
`PlanetSeeds[2]`), but this rests on two observations.

### SPEC-0043-REQ-010: Surface navigation
Distance between two points on a planet shall be the arc at the player's radius,
`|rel| * acos(u1.u2)`; bearing from the tangent of `u2` at `u1` in the frame of REQ-002; turn =
`((bearing - heading + 540) mod 360) - 180` (negative = left); height difference `|w| - |rel|`.

## How to re-measure (after a game update)

Every fact above is build-specific. Re-check them on a new build before trusting the map:

1. **Frame.** Stand inside a planetary base and compare `rel` with the save's base `Position`
   (expect tens of metres apart; the spread is your height in the base and the terrain).
2. **North/east.** Face due north on the in-game compass for ~5 s, then due east. Take the samples
   from those windows, project `-at` (on foot) onto the tangent plane, and compute the bearing with
   REQ-002: expect ~0 and ~90.
3. **`at` sign per location.** Over the trail, for consecutive samples > 3 m apart in the same
   location, compute `cos` between the step and `at`. Group by location: a mean near -1 means facing
   is `-at`, near +1 means `+at`. Mixed values (corvette interior, underwater) keep the on-foot sign.
4. **Radius.** Histogram `round(|rel| - distance_from_planet)` per planet; the mode should be a
   1024 multiple holding most samples.
5. **Sea level.** Visit a planet with oceans: `sealevel` non-zero; swim at the surface and check
   `|rel| - radius` is close to it.
6. **Address.** Attach from the main menu so `ua` is filled; pack it and compare with the save's
   `PlayerStateData.UniverseAddress` packed the same way.

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| No address and centre never seen with one | Not placed; "waiting for the game to report where you are". |
| Planet bits 0 / far from planet | Off-planet: show distance in km and bases in the same system, no map position. |
| No radius yet for a planet | Elevation and sector views say "not known until you visit this planet with the mod running". |
| Heading undefined (facing parallel to up) | Heading null; no arrow, no turn words. |
| Location not in the ship set but actually flying | Arrow reversed; add the location to the ship set after measuring as in REQ-003. |

## Verification

1. Stand in a planetary base: |rel - base Position| is tens of metres, not thousands.
2. Face north on the in-game compass: heading ~0; face east: ~90 (measured 89.4).
3. Walk north on the compass: `rel.y` rises and latitude rises (anywhere but at a pole).
4. Fly forward in the ship: the arrow points the way the trail grows (needs REQ-003's ship sign).
5. On a dry planet: `sealevel` is 0 and `|rel| - distance_from_planet` is a 1024 multiple for most
   samples. On a water planet: `sealevel` 120.
6. Pack a known save `GalacticAddress` from its fields with REQ-008 and compare to the save's integer.

## Lessons

See [SPEC-0049](../lessons/SPEC-0049-live-hook-lessons.md): ship heading reversed; "last radius wins"
gave a 24 m error; planet 0 treated as a planet; the corvette reports `Freighter`.
