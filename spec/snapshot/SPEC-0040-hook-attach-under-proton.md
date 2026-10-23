---
id: SPEC-0040
title: 'Attaching pyMHF / NMS.py to the running game under Proton'
description: 'Install a Windows Python inside the game''s Proton prefix, and a launcher that injects pyMHF into an already running NMS.exe exactly once per game run, with pre-flight checks and a safe detach.'
type: snapshot
area: live-hook
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: []
supersedes: []
superseded_by: []
related_specs: [SPEC-0041, SPEC-0042, SPEC-0047, SPEC-0049]
screenshots: []
---

# SPEC-0040: Attaching pyMHF / NMS.py to the running game under Proton

> **Safety.** This injects a Python interpreter into the running `NMS.exe` process and patches game
> functions. It can freeze or crash the game, and a crash during a save can damage that save.
> **Save in game first and keep a copy of the save folder** before every attach. Single-player only;
> never use it in an online / multiplayer session.

## Summary

No Man's Sky runs on Linux through Steam's Proton (Wine). The live map needs data that is not in the
save (player position several times a second, weather state, markers, star systems), so a Python mod
framework, **pyMHF**, with the game-specific layer **NMS.py** (`nmspy` on PyPI), is injected into the
running game. pyMHF is a Windows program: it has to run with a *Windows* Python installed *inside the
game's own Wine prefix*, using the *same Proton wine* as the game so that it joins the game's
wineserver and can see `NMS.exe` as a Windows process. A bash launcher (`nms-hook`) does every
pre-flight check, refuses the one situation known to kill the game (attaching twice), copies the mods
into place and execs pyMHF in attach mode. Once injected, pyMHF loads every `.py` in the mods folder;
the mods write JSONL files the exporter reads (SPEC-0042, SPEC-0044 to SPEC-0046).

## Inputs and outputs

| Thing | Where (Wine path / host path) |
|---|---|
| Game prefix | `<Steam library>/steamapps/compatdata/275850/pfx` (275850 = NMS's Steam app id) |
| Proton | `<Steam library>/steamapps/common/Proton - Experimental` (`files/bin/wine`); override with `NMS_HOOK_PROTON` |
| Windows Python | `C:\pymhf\python\python.exe` = `pfx/drive_c/pymhf/python/` |
| pyMHF local config | `C:\users\steamuser\AppData\Roaming\pymhf\nmspy\pymhf.local.toml` |
| Mods (loaded at inject) | `C:\pymhf\mods\*.py` (the launcher copies the repo's `pymhf_mods/*.py` here) |
| Mod output | `C:\pymhf\mod_saves\live\` = `pfx/drive_c/pymhf/mod_saves/live/` (format: `docs/hook-events.md`) |
| pyMHF logs | `C:\pymhf\logs\pymhf-YYYYMMDDTHHMMSS.log` |
| In-game executor | TCP `127.0.0.1:6770`, inside the game process (SPEC-0041) |

Versions in use (all must match each other and the game build):

| Component | Version |
|---|---|
| Game | Steam build 25800958, `NMS.exe` PE FileVersion **181442** |
| Windows Python | **3.12.10** amd64 (last 3.12 with a Windows installer) |
| nmspy | **181442.1** (nmspy's version number *is* the NMS.exe FileVersion it targets) |
| pyMHF | **0.2.5** (pinned by nmspy as `pymhf[gui,http_api]==0.2.5`) |
| Proton | Proton - Experimental (11.0 line), same one the game runs under |

## Requirements

### SPEC-0040-REQ-001: Windows Python in the game prefix
The installer shall install Windows Python 3.12.10 into `C:\pymhf\python` of the game prefix with the
game's Proton wine, unattended:
`wine python-3.12.10-amd64.exe /quiet InstallAllUsers=0 TargetDir=C:\pymhf\python PrependPath=0
Include_launcher=0 InstallLauncherAllUsers=0 Include_test=0 Shortcuts=0 AssociateFiles=0 Include_doc=0`
with `WINEPREFIX=<prefix> WINEDEBUG=-all WINEDLLOVERRIDES=winemenubuilder.exe=d`. The game shall not be
running during install. Proton's `files/bin/wine` works directly from the host; no `proton run`
wrapper is needed.

### SPEC-0040-REQ-002: nmspy install
The installer shall run `python.exe -m pip install --only-binary=:all: nmspy==<NMS.exe FileVersion>.<n>`
in the prefix Python. Every dependency (cyminhook, pymem, pyrun-injected, pywin32, dearpygui,
iced_x86, ...) is a prebuilt win_amd64 wheel; no compiler is needed.

### SPEC-0040-REQ-003: Version match
Before first use, the builder shall confirm that the installed nmspy targets the installed game:
nmspy's major version equals `NMS.exe`'s PE FileVersion. (An offline check that found 427 of 452 of
nmspy's byte signatures uniquely in `NMS.exe`'s `.text` was strong evidence of fit; only signatures for
functions a mod actually hooks matter.)

### SPEC-0040-REQ-004: Attach-only pyMHF config
The installer shall write `pymhf.local.toml` so that pyMHF never prompts and never starts the game:

```toml
[pymhf.local_config]
mod_dir = "C:\\pymhf\\mods"
mod_save_dir = "C:\\pymhf\\mod_saves"
start_exe = false
start_paused = false
[pymhf.local_config.logging]
log_level = "info"
window_name_override = "NMS.py"
log_dir = "C:\\pymhf\\logs"
shown = true
```

Without it pyMHF opens a folder chooser on first run, and with `start_exe = true` it tries to launch
the game through `steam://rungameid/275850`, which under Proton goes through xdg-open unpredictably.

### SPEC-0040-REQ-005: Find the game
The launcher shall find the game by scanning `/proc/*/comm` for `NMS.exe` and prefer the process whose
`/proc/<pid>/environ` has `WINEPREFIX=<prefix>` (with or without a trailing slash). If an `NMS.exe`
exists with another prefix it shall warn and use it; if none exists it shall exit 1 with "No Man's Sky
is not running ... start it from Steam, wait for the menu, then rerun".

### SPEC-0040-REQ-006: Refuse a second attach (port-6770 guard)
If `127.0.0.1:6770` is in LISTEN state in the game's network namespace, the launcher shall refuse to
inject and exit 1 with "port 6770 is already listening: this game was injected earlier. Restart the
game before attaching again." Detection: any line of `/proc/<game pid>/net/tcp` or `net/tcp6` whose
local address ends `:1A72` (6770 in hex) and whose state column is `0A` (LISTEN).
Rationale: pyMHF's injected side keeps 6770 until the game exits; a second attach fails with a memory
read error and pyMHF's `close_callback` then SIGTERMs the game (SPEC-0047, lessons).

### SPEC-0040-REQ-007: Same Proton, same wineserver, same sync mode
The launcher shall:
1. warn if the game's environment never mentions the Proton directory it will use (print the game's
   `STEAM_COMPAT_TOOL_PATHS`; suggest `NMS_HOOK_PROTON`);
2. warn if `/proc/<game>/ns/pid` differs from its own (game in a Steam Linux Runtime pid namespace);
3. copy every `WINE*SYNC*` and `PROTON_NO_*SYNC` variable from the game's environment into the
   environment of every wine call, so client and wineserver agree on esync/fsync/ntsync;
4. set `WINEPREFIX`, `WINEDEBUG=-all` (unless set), `WINEDLLOVERRIDES=winemenubuilder.exe=d`, and
   `WINEPATH=C:\pymhf\python;C:\pymhf\python\Scripts` (pyMHF starts its log window with
   `cmd /c start python ...`, so `python` must be on the Wine PATH).

### SPEC-0040-REQ-008: Prove the wine client sees the game
Before injecting, the launcher shall run the prefix Python with a snippet that lists Windows processes
(psutil) and prints `pymhf <ver> nmspy <ver> NMS.exe windows pid <n>`. If no Windows pid for `nms.exe`
appears, the launcher shall exit 1 ("this wine client is not on the game's wineserver") and inject
nothing. This catches a Proton mismatch or container isolation, where a second wineserver would start.

### SPEC-0040-REQ-009: `--check`
With `--check` the launcher shall run REQ-005 to REQ-008 and every install check (wine exists, prefix
exists, `python.exe`, `site-packages/nmspy`, `site-packages/pymhf`, the local config exists and
matches `^start_exe *= *false`), print the results and exit 0 without injecting. `--check` shall not
need a tty.

### SPEC-0040-REQ-010: Real terminal and an explicit "saved?" answer
When injecting, the launcher shall require stdin and stdout to be a tty (pyMHF imports questionary /
prompt_toolkit at import time, which raises `NoConsoleScreenBufferError` without a console), and shall
ask "This injects Python into the running game and may crash it. Have you saved? [y/N]", injecting
only on `y`/`yes`. `--yes` skips the question.

### SPEC-0040-REQ-011: Install mods and exec pyMHF
The launcher shall copy `pymhf_mods/*.py` into `C:\pymhf\mods\`, `cd` to `drive_c/pymhf`, and `exec`
`wine C:\pymhf\python\python.exe -m pymhf run nmspy` with the environment of REQ-007. pyMHF then
injects (pyrun-injected remote-thread LoadLibrary of `python312.dll` into `NMS.exe`), loads the mods,
hooks via MinHook, and serves the executor on `127.0.0.1:6770`. Two extra windows appear: the pyMHF
log console and the DearPyGui window.

### SPEC-0040-REQ-012: `--detach`
With `--detach` the launcher shall SIGKILL every host process whose command line contains `C:\pymhf\`
(launcher python, `start.exe`, the log terminal), print them, and say that the injected code stays in
the game until it exits. It shall use SIGKILL, never SIGINT/SIGTERM, so pyMHF's exit path (which kills
the game) never runs. The game keeps running and the mods keep writing.

### SPEC-0040-REQ-013: Never Ctrl+C
The launcher's help shall say: do not Ctrl+C the pyMHF terminal; detach with `--detach`. In pyMHF
0.2.5 the end of the injected script for any reason fires `close_callback`, which does
`os.kill(game_pid, SIGTERM)` regardless of `start_exe`.

### SPEC-0040-REQ-014: Mod import check before shipping a mod
Where a mod is changed, the builder shall import it with the prefix Python before attaching or
hot-loading it: under a pseudo-tty (`script -qec "<cmd>"`), with `WINEPREFIX`, the game's sync
variable (e.g. `WINEFSYNC=1`) and the game's Proton wine.

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| Game not running | Launcher exits 1 before anything Windows-side runs. The mods write nothing; `live.json` goes stale and the map shows "stale" after `STALE_S = 10` s (SPEC-0042). |
| Prefix Python / nmspy / pymhf / local config missing | Launcher exits 1 naming the missing piece. |
| Wine client cannot see `NMS.exe` | Exit 1, nothing injected. Fallback (untried): start the game with `STEAM_COMPAT_LAUNCHER_SERVICE=proton %command%` and run the same command inside the container with `steam-runtime-launch-client --alongside-steam --appid 275850 -- ...`. |
| Game already injected this run (6770 listening) | Refuse; tell the player to restart the game. The earlier injection is still serving, so the hot-patch route (SPEC-0041) is the way to change mods. |
| Not a tty | Inject path exits 1; `--check` still works. |
| Injected side errors | pyMHF terminates the game. Nothing in the launcher can prevent this; it is why saving first is mandatory. |
| A mod fails to import | pyMHF logs it and loads the others; that mod's files stay stale. |
| A detour raises | pyMHF disables that one detour, the game survives; see SPEC-0041 for re-enabling. |
| Host side killed (`--detach`, logout, terminal closed by SIGKILL) | Game and mods keep running; no new attach possible until the game restarts. Closing the terminal normally (SIGHUP) is untested: treat it as Ctrl+C. |

## Verification

1. Game running, menu or loaded save: `nms-hook --check` prints
   `wine side: pymhf 0.2.5 nmspy 181442.1 NMS.exe windows pid <n>` and `--check: not injecting.`
   (observed: Windows pid 332).
2. `--check` with the game stopped exits 1 with "not running"; with a fake `STEAM_ROOT`, exits 1 with
   "Windows Python not installed in the prefix"; an unknown flag exits 2.
3. Attach: the pyMHF log shows "Python injected", "Found PID", "Exe hash is ...", "pyMHF injection
   complete!", "Loaded N mods and M hooks", "Serving on executor ('127.0.0.1', 6770)"; the game keeps
   running; `mod_saves/live/live.json` starts updating every 0.25 s.
4. Port guard: with a test listener on 127.0.0.1:6770 in your own netns, the awk test over
   `/proc/$$/net/tcp` reports held; without it, free. Then `nms-hook` against an injected game refuses.
5. `nms-hook --detach`: listed processes die, the game keeps running, `live.json` keeps updating.

## Lessons

See [SPEC-0049](../lessons/SPEC-0049-live-hook-lessons.md). Top two: a second attach into an
already-injected game crashed it (hence REQ-006); Ctrl+C or any end of the injected script SIGTERMs
the game (hence REQ-012/013).
