---
id: SPEC-0049
title: 'live-hook lessons'
description: 'What hooking the running game under Proton cost to learn: game-killing attach and exit paths, a hot-patch that crashed the game, a disabled detour, reversed headings, wrong radii and storm tables, and smaller traps.'
type: lesson
area: live-hook
author: bob
created: 2026-10-23
last_updated: '2026-10-23'
status: current
related_specs: [SPEC-0040, SPEC-0041, SPEC-0042, SPEC-0043, SPEC-0044, SPEC-0045, SPEC-0046, SPEC-0047, SPEC-0048]
---

# SPEC-0049: live-hook lessons

Ordered by how much time (and game progress) each would save a second builder. All dates 2026-10-22
unless stated; build 25800958, nmspy 181442.1, pyMHF 0.2.5.

## Lesson: a second attach into the same game run crashed the game

**What happened.** After the first successful attach, the host side was killed and `nms-hook` was
run again against the same running game to load a changed mod. pyMHF logged
`Could not read memory at: 5931466752, length: 24 - GetLastError: 5`, then "pyMHF exiting...", and
the game closed. The injected side of the first attach was still in the game holding port 6770.
Saves written 1.5 min earlier decoded fine; nothing else was lost, by luck of timing.

**The rule.** One attach per game run. The launcher checks `/proc/<game>/net/tcp{,6}` for a LISTEN
socket on `:1A72` (6770) and refuses (SPEC-0040 REQ-006). Change mods through the executor
(SPEC-0041) or restart the game.

## Lesson: Ctrl+C in the pyMHF terminal ends the game, and SIGINT does nothing

**What happened.** The first plan said "Ctrl+C detaches". Reading `pymhf/main.py` showed
`close_callback`, fired whenever the injected script's future completes, SIGTERMs the game's pid in
every mode; Ctrl+C sends the injected side its exit sequence, so it would close the game. Trying
SIGINT on the host-side Windows python instead did nothing at all (Wine does not map a Unix SIGINT to
a console Ctrl+C), which looked like "detach is broken".

**The rule.** Never Ctrl+C, never close the pyMHF terminal politely. Detach with SIGKILL of every host
process whose command line contains `C:\pymhf\` (`nms-hook --detach`). The game and mods keep running.

## Lesson: a hot-patch that read a new game object froze and crashed the game

**What happened.** (2026-10-23) Attached mid-game on foot, nmspy had no application object (it finds
it only on a game state change), so `ua` was null and the map waited for an address. A hot-patch added
a `cGcApplication.Update.after` detour that did
`gameData.GcApplication = map_struct(get_addressof(this), nms.cGcApplication)` and let the existing
address code read through it. The game froze within seconds and crashed; the last live sample was
14 s after the last save. The repo change (`live_position` 0.5) was reverted to 0.4.

**The rule.** Live patches may only change Python logic over fields the running mods already read
without incident. New memory reads (new struct mappings, pointers, offsets, hooks) go into the mod
file for the next attach, after the player saves. A feed that freezes right after a patch is a hang,
not a pause. For the missing address, infer it in the exporter from the planet centre seen earlier
with an address (SPEC-0043 REQ-008).

## Lesson: one odd enum byte got the marker detour disabled for the session

**What happened.** `cGcMarkerPoint.meBuildingClass` read 63 (`'?'`) on most markers; that is not a
`cGcBuildingClassification` member. The mod's enum fallback called `.name` on the `c_enum8` field,
which raised; pyMHF logged "There was an error with detour <bound method VisorLog.on_add_marker ...>.
It has been disabled." and `marker_add` events silently stopped while visor events carried on. Nothing
else signals it: the game is fine and the other detours keep writing.

**The rule.** Decode enums to the name *or the raw int* (one-byte `bytes` -> first byte -> int ->
`Enum(raw).name`, `ValueError` -> raw). Wrap every detour body; log the first failure once as an event
and never raise (SPEC-0045 REQ-005/006). After fixing, move the detour from `_disabled_detours` back
to its list from the executor (SPEC-0041 REQ-005). Check the hook census after every attach.

## Lesson: the ship's `at` points forward, not back

**What happened.** On foot, movement was opposite `mPlayerTM.at` (cos -0.96 to -1.00), so facing was
taken as `-at` everywhere. In flight the map's arrow and the "turn left/right" words were reversed.
Over the trail: on foot cos -0.94 (304 steps); in the ship +0.99 (1,042 steps), locations
`PlanetInShip` and `Default` (the ship in space reports `Default`, not a ship-specific value).

**The rule.** Facing = `+at` in `PlanetInShip` and `Default`, `-at` elsewhere (SPEC-0043 REQ-003).
Measure the sign per location from trail steps (movement vs `at`) before trusting an arrow.

## Lesson: storms on an Extreme planet ran far longer than the Low table

**What happened.** The planet's `StormFrequency` was Low, so the countdown used Low: 120-180 s storms,
900-2400 s gaps. The logged storm lasted 315.0 s (Active -> TransitionOut), and the next one came
359.2 s after the first cleared, without Warning or TransitionIn (player in the corvette). The
countdown was confidently wrong twice.

**The rule.** Intensity Extreme overrides frequency: High length (150-320 s) and the ExtremeFallback
gap (240-620 s). Storm length excludes the 20 s transitions. Log every transition with a timestamp
so timing tables can be checked against reality (SPEC-0044 REQ-004).

## Lesson: "last radius wins" put the ground 24 m off

**What happened.** The planet's nominal radius was taken from the latest sample's
`|rel| - distance_from_planet`. Most samples give exactly a 1024 multiple (818 of 1,002 gave 98304 on
one planet), but samples in the ship or underwater gave 98280-98303, so heights jumped by up to 24 m
depending on how the player last moved.

**The rule.** Use the most common value per planet (SPEC-0043 REQ-005). Expect a multiple of 1024.

## Lesson: planet index 0 is not a planet

**What happened.** Address planet bits 0 means "in this system, at no planet" (space, galaxy map).
Early code decided "on a planet" from `in_atmosphere` or `distance_from_planet <= 20000` alone, so
samples near a planet but with planet bits 0 (e.g. just after arriving in a system, or with the galaxy
map open) were recorded as a planet of their own: it got a nominal radius, a map anchor and an entry
in the visited-planets list.

**The rule.** Never treat planet bits 0 as a planet: no radius, no map, no name (SPEC-0043 REQ-007).
Portal codes turn planet 0 into 1.

## Lesson: the corvette interior reports location "Freighter"

**What happened.** Inside the player's corvette the game reports `EnvironmentLocation.Freighter` (9);
the capital freighter is `FreighterInternals` (10). The map said "Freighter" while the player stood in
their corvette on a planet. Separately, the corvette and freighter are *bases* in the save, so a planet
with the corvette parked on it was taken to "have bases" and dropped from the base-less planet list.

**The rule.** Label 9 as the corvette, 10 as the freighter (SPEC-0043 REQ-004); count only planetary
bases (exclude FreighterBase, PlayerShipBase, PlayerSpaceBase) when asking "does this planet have
bases".

## Lesson: the hotspot class is not in the marker

**What happened.** An exploratory pass went into finding which byte of a hotspot marker carries C/B/A/S: the mod
dumped every hotspot marker's 0x2B0-byte struct as hex. A class B power marker and a class S gas
marker differed in 26 bytes, all explained by position, label and type.

**The rule.** The player records the class (`/hs S silver` in chat, or on the map). Do not search the
marker for it again; if anything, look in the visor's scan UI path (not done).

## Lesson: pyMHF needs a real console even to import

**What happened.** `python -c "import pymhf"` with stdout piped failed:
`prompt_toolkit.output.win32.NoConsoleScreenBufferError: Found xterm-256color, while expecting a
Windows console` (questionary prompts are built at import time).

**The rule.** Run imports and the launcher's inject path under a tty (`script -qec "..."` for
scripted checks). `--check` avoids importing pymhf so it works without one.

## Lesson: some hooks fire thousands of times; dedupe at the source

**What happened.** `SpacePoi` markers re-added every frame (879 of the first 902 visor lines);
`GeneratePlanetName` ran every frame for the current planet; `Classify` runs for thousands of systems
per frame on the galaxy map. The files ballooned within minutes.

**The rule.** Mute churn types, key markers by (event, type, 1 m position) with a 60 s repeat, log
names once per (seed, name), and make Classify return on a set lookup before anything else
(SPEC-0045 REQ-007, SPEC-0046 REQ-001/003).

## Lesson: a mid-game attach may never learn the address

**What happened.** nmspy fills `gameData.player_state` only on a game state change (load, warp), so
attaching while walking on a planet left `ua` null indefinitely. The tempting fix is the crash above.

**The rule.** Attach from the main menu or right after loading where possible; otherwise let the
exporter infer the address from a planet centre seen before with one, and say "waiting for the game to
report where you are" when it cannot.

## Lesson: attach mode was the root of the "no address" problem; launching fixes it

**What happened.** For two days the exporter inferred the planet from its centre and the map
waited for an address after every mid-game attach, and one hot-patch to get the application
object another way crashed the game. The cause was structural: nmspy captures `cGcApplication`
with a one-shot hook on the FSM's first state change *after injection*, which never comes when
you attach to a game already in APPVIEW. Launching through pyMHF (SPEC-0040 launch mode) put
injection 4 s before the first state, and the first sample had the address.

**The rule.** When a singleton is `None` after attaching, don't go looking for another way to
read it; inject earlier. Reading the framework's own singleton loader tells you which hook it
needs and whether you missed it.
