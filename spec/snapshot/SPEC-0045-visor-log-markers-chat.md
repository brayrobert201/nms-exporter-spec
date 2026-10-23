---
id: SPEC-0045
title: 'visor_log mod: analysis visor, game markers, hotspots and chat commands'
description: 'pyMHF mod logging visor open/close and scans, every marker the game adds or removes (deduplicated, noisy types muted, hotspot structs as raw hex), and in-game chat commands /hs /mark /help that record hotspot classes and named points.'
type: snapshot
area: live-hook
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0040, SPEC-0043]
supersedes: []
superseded_by: []
related_specs: [SPEC-0041, SPEC-0044, SPEC-0048, SPEC-0049]
screenshots: []
---

# SPEC-0045: visor_log mod: analysis visor, game markers, hotspots and chat commands

> Runs inside the game process (SPEC-0040). Save before attaching.

## Summary

Extraction hotspots (power, mineral, gas) and resource deposits are not in the save in a usable form,
but the game places a marker for each when the player finds it. `visor_log.py` hooks the analysis visor
and the game's marker list and logs, as events, every marker the game adds or removes, with its type,
label and position, so the map can draw "finds". The hotspot's class (C/B/A/S) is **not** in the
marker, so the mod also adds in-game chat commands: the player types `/hs S silver` while standing on
a hotspot and the class and resource are recorded at their position. Output:
`visor-YYYYMMDD.jsonl` (fields: `docs/hook-events.md`).

## Inputs and outputs

| Hook (nmspy name) | When | Event(s) |
|---|---|---|
| `cGcBinoculars.UpdateTarget.after(this, lfTimeStep)` | every frame while the visor is up | `visor_open` on first call |
| `cGcPlayerEnvironment.Update.after` | every frame | `visor_closed` when UpdateTarget idle > `VISOR_IDLE_S = 0.75` s; refreshes the player context once per 1.0 s |
| `cGcBinoculars.UpdateScanBarProgress.after(this, lfScanProgress)` | scan bar | `scan_start`, `scan_complete`, `scan_abandoned` |
| `cGcBinoculars.PopulateDiscoveryInfo.after(this, lDiscoveryInfo, lpTargetNode, lpTargetAttachment, lbSubmitDiscovery, lbSilent)` | a scan result filled in | `discovery_info` |
| `cGcBinoculars.SetMarker.after(this)` | player tags with the visor | `visor_tag` |
| `cGcMarkerList.TryAddMarker.before(this, lPoint, lbUpdateTime)` | game adds a marker | `marker_add` |
| `cGcMarkerList.RemoveMarker.before(this, lExampleMarker)` | game removes a marker | `marker_remove` |
| `cGcTextChatInput.ParseTextForCommands.before(this, lMessageText)` | player sends a chat line | `label`, `mark`, `chat_error` |

All hooks matched NMS.exe 181442 in the offline signature scan.

## Requirements

### SPEC-0045-REQ-001: Player context
Once per second the mod shall store `player = {pos, planet_pos, location, at}` from
`cGcPlayerEnvironment` (2-decimal vectors) and attach it to every event row as `player`.

### SPEC-0045-REQ-002: Visor open/close
On the first `UpdateTarget` after the visor was closed, write `visor_open`. In the per-frame
environment hook, if the visor is open and `now - last UpdateTarget > 0.75` s, write `visor_closed`.

### SPEC-0045-REQ-003: Scan bar
With `p` the new progress and `last` the previous: `last <= 0 < p` -> `scan_start`;
`p >= 1 > last` -> `scan_complete`; `p <= 0 < last < 1` -> `scan_abandoned` with `reached = last`.

### SPEC-0045-REQ-004: Marker rows
For each marker pointer (non-null), read the inline `cGcMarkerPoint` fields:
`meType` (`cGcMarkerType`), `meBuildingClass` (`cGcBuildingClassification`), `mCustomName`,
`mCustomSubtitle` (strings, NUL/space-stripped, null if empty), `mPosition.local` and
`mPosition.offset` (a `cTkPhysRelVec3`; world = local + offset). Write `type`, `building`, `name`,
`subtitle`, `local`, `offset`; and when the player context exists, `rel = world - player.planet_pos`
and `dist_from_player = |world - player.pos|`.
`rel` is the planet frame of SPEC-0043: a `PlayerBase` marker sat 4.6 m above its base's saved
Position.

### SPEC-0045-REQ-005: Safe enum decode
Every enum field shall be decoded by: take the value (an int, or a `c_enum8` whose `.value` is a
one-byte `bytes` - use its first byte), convert to int, return `Enum(raw).name`, or the raw int if the
enum has no such member. Never call `.name` on the field object directly: it raises for unknown values.
Observed: `meBuildingClass` reads 63 (`'?'`) on most markers, which is not a
`cGcBuildingClassification` value.

### SPEC-0045-REQ-006: A bad marker never disables the detour
Marker handling shall run inside `try/except Exception`; on the first failure in a session, log the
traceback and write one `marker_error` event with `error = repr(e)`; later failures are silent. The
detour shall never raise (pyMHF would disable it: SPEC-0041 REQ-005).

### SPEC-0045-REQ-007: Dedup and muted types
The mod shall drop markers whose type is in
`QUIET_TYPES = {NetworkPlayer, NetworkPlayerShip, NetworkPlayerVehicle, NetworkPlayerFireTeam,
NetworkPlayerFireTeamShip, NetworkPlayerFireTeamFreighter, SpacePoi}` (SpacePoi moved every frame:
879 of the first 902 lines). Others are keyed by `(event, type, world position rounded to 1 m)` and
logged at most once per `MARKER_REPEAT_S = 60` s per key.

### SPEC-0045-REQ-008: Raw hotspot structs
For marker types `PowerHotspot` (0x38), `MineralHotspot` (0x39), `GasHotspot` (0x3A), the row shall
add `raw`: the whole `MARKER_SIZE = 0x2B0`-byte marker struct as lowercase hex
(`ctypes.string_at(addressof(m), 0x2B0)`). Kept for checking other fields; it did not reveal the class.

### SPEC-0045-REQ-009: Hotspot facts the map relies on
- A hotspot marker appears when the player **arrives at the point a survey led to**, not when the
  survey starts. Its label is the survey label (e.g. `UI_SURVEY_MINERALS_LABEL`).
- **The class is not in the marker.** A class B PowerHotspot and a class S GasHotspot marker differed
  only in position, label and type (26 of 0x2B0 bytes). The player reads the class in the visor and
  records it (REQ-010 or the map).
- A **mineral** hotspot can give any of the planet's three substances (seen: the common one on one
  planet, the rare one on another); the marker does not say which, so the map lists all three until
  the player records it.
- **Gas**: Gas1 is set by biome (Lush, Toxic, Blue, Swamp -> Nitrogen; Radioactive, Frozen, Green ->
  Radon; Scorched, Barren, Red, Lava -> Sulphurine; Waterworld, Gas Giant -> Methane; Dead, Weird ->
  Oxygen); Gas2 is always Oxygen. Seen: a Toxic planet's class S gas hotspot gave Nitrogen.
- Deposits arrive as `TerrainResource` markers named after the substance ("Cobalt Deposit").
- Marker rows carry no address; the exporter assigns them to the planet the trail says the player was
  on at time `t`.

### SPEC-0045-REQ-010: Chat commands
In `ParseTextForCommands.before`, read the line (`str(lMessageText.contents).strip()`), and:
- if it is exactly `/help`, `/hs` or `/mark` (case-insensitive), or starts with `/help `, replace the
  line with the help text and record nothing;
- if the first whole word is `/hs`: from the remaining words, `class` = the first word that is one of
  `C B A S` (case-insensitive, upper-cased), `resource` = the other words joined by spaces (null if
  none); write `label` with `rel = player.pos - player.planet_pos` (null if no context yet), `class`,
  `resource`; reply "Noted for the nearest hotspot: class S silver";
- if the first word is `/mark`: `name` = the rest of the line (default `mark`); write `mark` with
  `name`, `rel`; reply "Marked 'name' on the map";
- if `rel` is null, append " (position not known yet: try again in a second)";
- replace the chat line with the reply via `lMessageText.contents.set(reply)` so the game posts the
  reply instead of the command;
- any other line passes through untouched. Errors: log and write `chat_error`, never raise.
Help text uses the game's chat markup `<TITLE>...<>`:
`/hs S silver` class and/or resource of the hotspot you stand on (either can be left out);
`/mark Ammonia field` a named flag at your position on the map; `/help` this list.

### SPEC-0045-REQ-011: Exporter use of labels
The exporter shall attach a `label` to the nearest hotspot within `CHAT_LABEL_M = 50` m on the same
planet; a later label set on the map wins per field over an older chat label. Finds are keyed by
`(address, kind, round(rel / MERGE_M))` per axis with `MERGE_M = 5` m, so repeat sightings of one
hotspot merge.

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| Mod not attached | No new finds; existing ones still shown from history. |
| Marker detour disabled | `marker_add` lines stop while `visor_open` etc. continue: the census (SPEC-0041) shows it; re-enable. |
| No player context yet (first second) | Marker rows lack `rel`/`player`; the exporter places them only if it can. |
| Chat before position known | Event written with `rel: null`; reply says to retry. |
| Unknown enum values | Raw int in the field (e.g. `building: 63`). |

## Verification

1. Open and close the visor: one `visor_open` then one `visor_closed` ~0.75 s after closing.
2. Scan a plant: `scan_start` then `scan_complete`; `discovery_info` with `submit`, `silent`.
3. Follow a survey to a hotspot: `marker_add` of `MineralHotspot`/`GasHotspot`/`PowerHotspot` with
   `raw`, `dist_from_player` small.
4. Type `/hs S silver` on it: chat shows the reply; a `label` line with `class: "S"`,
   `resource: "silver"`; the map labels the hotspot.
5. Type `/hs` alone: help appears; nothing written.

## Lessons

See [SPEC-0049](../lessons/SPEC-0049-live-hook-lessons.md): `meBuildingClass` read 63 and the enum
fallback raised, so pyMHF disabled the marker detour; the class is not in the marker.
