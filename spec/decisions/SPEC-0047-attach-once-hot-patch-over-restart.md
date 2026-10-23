---
id: SPEC-0047
title: 'Attach once per game run; change mods by hot-patching through the executor'
description: 'The launcher refuses a second attach and detaches by SIGKILL; mod changes reach a running game through pyMHF''s executor, limited to Python logic over fields already proven safe.'
type: decision
area: live-hook
author: bob
created: 2026-10-23
last_updated: '2026-10-23'
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0040, SPEC-0041, SPEC-0049]
---

# SPEC-0047: Attach once per game run; change mods by hot-patching through the executor

## Context

pyMHF 0.2.5 injects a Python interpreter into `NMS.exe`. Three facts of that version, all observed
or read from its source on 2026-10-22, constrain how it can be used:

1. **The injected side never leaves.** It holds `127.0.0.1:6770` and its hooks until the game exits.
   There is no unload.
2. **A second attach into the same game run fails and kills the game.** It failed with
   `Could not read memory at: 5931466752, length: 24 - GetLastError: 5`, printed "pyMHF exiting...",
   and the game closed.
3. **Any end of the injected script kills the game.** `close_callback` in `pymhf/main.py`, attached to
   the future that waits on the injected script, does `os.kill(game_pid, SIGTERM)` without checking
   `start_exe`. Ctrl+C in the pyMHF terminal sends the exit sequence to the injected side, so it ends
   the game too (read from source; not tested deliberately). SIGINT to the Windows python from the host
   does nothing: Wine does not turn a Unix signal into a console Ctrl+C.

Meanwhile the mods were exploratory and changed many times an evening, and a game restart costs
minutes plus the risk to unsaved progress each time.

## Decision

> **Scope (2026-10-23).** This decision governs *attach mode*. Launch mode (SPEC-0040, "Launch
> mode") is now the normal way in: every session is a fresh launch with injection at boot, so
> "one attach per run" is not a rule anyone has to remember, mid-game attaches with no address do
> not happen, and Ctrl+C in pyMHF's console is simply how the game is closed. The executor
> hot-patching below applies unchanged in both modes. Attach mode remains for a game that is
> already running (started without the wrapper) and keeps every rule here.


- **One attach per game run.** The launcher detects a listening 6770 in the game's network namespace
  and refuses to inject (SPEC-0040 REQ-006). Restarting the game is the only way to attach again.
- **Detach = SIGKILL the host side** (`nms-hook --detach`), never Ctrl+C, so no exit path runs. The
  injected code keeps running and writing.
- **Mod changes go through the in-game executor** (SPEC-0041): replace functions on the mod class,
  mutate constants, re-enable detours pyMHF disabled, `load_single_mod` for new mods. The repo copy of
  each mod is updated and copied into `C:\pymhf\mods` first, so the next attach runs the same code.
- **Hot-patches change Python logic only, over fields the running mods already read safely.** Any
  change that reads new game memory (a new struct mapping, a pointer, a new hook) waits for the next
  attach, after the player saves and restarts.

## Alternatives considered

- **Restart the game for every mod change.** Safe but slow; would have turned an evening of iteration
  into a handful of attempts, and every restart is a chance to lose unsaved progress.
- **Patch pyMHF's `close_callback`** in the prefix's site-packages to skip the SIGTERM in attach mode.
  Would make Ctrl+C and injected-side errors survivable, but edits a third-party package that `pip`
  overwrites on upgrade, and does not make a second attach work (the memory-read failure comes first).
  Worth revisiting upstream rather than locally.
- **pyMHF's DearPyGui / HTTP API** for control. The GUI is per-mod widgets, not code reload; the HTTP
  API exposes only endpoints mods declare. The executor is general.
- **Re-inject after unloading.** pyMHF 0.2.5 has no unload.

## Consequences

- Iteration on mods is seconds, not minutes; the marker-detour fix, the SpacePoi mute, the raw
  hotspot dump, biome fields and the whole `galaxy_log` mod all went into one running game.
- The running game can drift from the repo if a patch is not also written to the file; the rule
  "edit the file, compile, copy, then patch from the file" keeps them equal.
- The executor is an unauthenticated code-execution port on loopback for as long as the game runs.
  Acceptable on a single-user gaming PC; not on a shared machine.
- The memory-read rule was learned by crashing the game once (2026-10-23, `cGcApplication::Update`
  `this` mapped as `cGcApplication`). Mid-game attaches therefore may lack the galactic address; the
  exporter infers it from the planet centre (SPEC-0043 REQ-008). The real fix was earlier injection
  (launch mode), not another read.
- Revisited 2026-10-23: launch mode removes the second-attach trap and the missing address for normal
  play; this decision stays for the attach fallback. Revisit again when pyMHF supports
  unload/re-attach, or stops terminating the game on injected-side exit.
