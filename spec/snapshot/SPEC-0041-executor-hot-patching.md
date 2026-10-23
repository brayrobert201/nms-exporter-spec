---
id: SPEC-0041
title: 'In-game executor and hot-patching mods without a restart'
description: 'Sending Python to pyMHF''s executor on 127.0.0.1:6770 to inspect hooks, replace mod functions, re-enable disabled detours and load new mods into a running game, and the rule that live patches never read new game memory.'
type: snapshot
area: live-hook
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0040]
supersedes: []
superseded_by: []
related_specs: [SPEC-0042, SPEC-0044, SPEC-0045, SPEC-0046, SPEC-0047, SPEC-0049]
screenshots: []
---

# SPEC-0041: In-game executor and hot-patching mods without a restart

> **Safety.** Code sent to the executor runs inside the game process with no sandbox. A wrong read of
> game memory can freeze and crash the game. Save first; patch only Python logic (REQ-007).

## Summary

A game run can be attached only once (SPEC-0040, SPEC-0047), so every mod change would otherwise cost
a game restart. pyMHF's injected side runs an **executor**: a TCP server on `127.0.0.1:6770` inside the
game that `exec`s whatever Python text it receives and sends anything `print`ed back on the same
socket. Through it a running mod can be inspected, its functions replaced, a detour pyMHF disabled can
be put back, and a whole new mod file loaded. This is what made iterating on the mods practical. The
one hard limit: a live patch may only change Python logic over game fields the running mods already
read safely. Anything that reads new game memory goes into the mod file for the next attach.

## Inputs and outputs

- In: Python source text over TCP to `127.0.0.1:6770` (loopback only; any local process can connect).
- Out: whatever the code `print`s, or a traceback, written back on the socket as UTF-8.
- Side effects: the code runs in the injected interpreter's module globals, which persist between
  messages.

## Requirements

### SPEC-0041-REQ-001: Wire protocol
The executor client shall open a TCP connection to `127.0.0.1:6770`, send the Python source as UTF-8
with `sendall`, half-close (`shutdown(SHUT_WR)`), then read until EOF or a 10 s timeout and print what
came back. Facts of pyMHF 0.2.5's server (`pymhf/injected.py`, `ExecutingProtocol`):
- each received data chunk is passed to `exec(data.decode())` on its own. Keep a message small enough
  to arrive in one chunk (a few kB is fine on loopback); for anything large, write a file into the
  prefix and send `exec(open(r"C:\pymhf\patch.py").read())`.
- `print` is rebound to write to the socket; an exception prints its traceback to the socket.
- the snippet's locals are copied into the executor module's globals after a successful run. Prefix
  every name in a snippet with `_` (`_mm`, `_hm`, `_inst`) so nothing shadows real globals.
- two magic byte sequences (pyMHF's `ESCAPE_SEQUENCE` and `READY_ASK_SEQUENCE`) end the executor or
  ask readiness. Never send `ESCAPE_SEQUENCE`: ending the injected side SIGTERMs the game.

### SPEC-0041-REQ-002: Handles available in the game
Snippets shall reach the running state through:
- `from pymhf.core.mod_loader import mod_manager` - `mod_manager.mods` is a dict class-name ->
  instance (`"LivePosition"`, `"VisorLog"`, `"GalaxyLog"`); `sys.modules[type(inst).__module__]`
  is the mod's module (for module-level functions and constants).
- `from pymhf.core.hooking import hook_manager` - `hook_manager.hooks` maps a function identifier
  (`.name` like `"cGcMarkerList.TryAddMarker"`) to a FuncHook with `state`, `_before_detours`,
  `_after_detours`, `_after_detours_with_results`, `_disabled_detours` (a set);
  `hook_manager._uninitialized_hooks`, `hook_manager.failed_hooks`.
- `from nmspy.common import gameData` - nmspy's globals (`player_state`, `GcApplication`).

### SPEC-0041-REQ-003: Inspect before patching
Before any patch, the operator shall list the hooks the change touches, with state and the names in
each detour list, and the mod instance's relevant attributes (e.g. `LivePosition._next` versus
`time.time()`, to tell a hook that stopped firing from a mod that stopped writing).

### SPEC-0041-REQ-004: Replace a mod function
To change mod logic live, the operator shall:
1. edit the repo file, `py_compile` it, and copy it into `C:\pymhf\mods\` (so the next attach gets the
   same code);
2. extract the changed functions' source from the file (`ast.get_source_segment`), and send a snippet
   that `exec`s each module-level function into the mod module's `__dict__`, and each method into a
   scratch namespace then assigns it onto the mod *class* (`type(inst).name = ns["name"]`). The bound
   detours pyMHF holds call through the class, so the new method takes effect on the next call;
3. reset any rate-limit state on the instance so the effect shows at once (e.g. `_planet_next = 0`);
4. for a changed constant, mutate it in place (`module.QUIET_TYPES.add("SpacePoi")`).
A changed `__init__` does not re-run; add new instance attributes from the snippet.

### SPEC-0041-REQ-005: Re-enable a detour pyMHF disabled
When a detour raises, pyMHF 0.2.5 pops it from its list, logs "There was an error with detour <bound
method ...>. It has been disabled." and adds it to `_disabled_detours`. (In the *before* list the
exception also skips the remaining before-detours for that call.) After patching the cause (REQ-004),
the operator shall, for that hook, move each detour from `_disabled_detours` back to the list it came
from (`_before_detours` for a `.before` hook, `_after_detours` for `.after`) and print what moved.
pyMHF has no public API for this in 0.2.5.

### SPEC-0041-REQ-006: Load a new mod live
To add a mod to a running game, the operator shall copy the file into `C:\pymhf\mods\` and send
`mod_manager.load_single_mod(r"C:\pymhf\mods\<file>.py")`, which returns `(mods, hooks)` loaded and
binds the hooks. Check `"<ClassName>" in mod_manager.mods` and the pyMHF log for errors, and watch the
mod's output file appear. (Used for `galaxy_log.py` on 2026-10-22.)

### SPEC-0041-REQ-007: Live patches never read new game memory
A live patch shall only change Python logic over struct fields that the running, attached mods have
already been reading without incident. A patch that maps a new game object, follows a new pointer,
reads a new offset or hooks a new function shall not be sent to a running game; it goes into the mod
file for the next attach, after the player has saved and restarted the game.
Origin: on 2026-10-23 a hot-patch added a `cGcApplication::Update` detour that mapped its `this` as
`cGcApplication` and read the galactic address through it; the game froze within seconds and crashed
(last sample 14 s after the last save). The same change was reverted in the mod (lessons).

### SPEC-0041-REQ-008: Mods are written to be patchable
Every mod shall:
- read only fields stored inline in the structs passed to its detour (`this.contents.<field>`, nested
  inline structs, inline arrays); never follow a pointer;
- never let an exception escape a detour: wrap per-call work in `try/except`, log the first failure
  per hook once (to the pyMHF log and as an `error`/`marker_error`/`chat_error` event in its JSONL),
  and carry on;
- keep its state on the instance and its tunables as module constants, so REQ-004 can reach them.

### SPEC-0041-REQ-009: Frozen feed after a patch means hang
If `live.json` stops updating within seconds of a patch while the game was in play, the operator
shall treat it as a hang, not a pause, and tell the player to save (if the game still responds) and
restart rather than send more code.

## Reference snippets

These are the spec: the client and the two non-obvious patch shapes.

Client (host side, any Python 3; reads code on stdin):

```python
import socket, sys
s = socket.create_connection(("127.0.0.1", 6770), timeout=10)
s.sendall(sys.stdin.read().encode())
s.shutdown(socket.SHUT_WR)
out = b""
try:
    while (b := s.recv(65536)):
        out += b
except socket.timeout:
    pass
print(out.decode(errors="replace"))
```

Replace methods and re-enable a disabled before-detour (sent to the executor; `_src_*` are the
function sources pasted in by the generator):

```python
import sys as _s
from pymhf.core.mod_loader import mod_manager as _mm
from pymhf.core.hooking import hook_manager as _hm
_inst = _mm.mods["VisorLog"]
_mod = _s.modules[type(_inst).__module__]
exec(_src_module_fn, _mod.__dict__)              # module-level helper, e.g. _name
_ns = {}
exec(_src_methods, _mod.__dict__, _ns)           # methods, dedented to top level
type(_inst)._marker_row = _ns["_marker_row"]
for _k, _h in _hm.hooks.items():
    if _k.name == "cGcMarkerList.TryAddMarker":
        for _d in list(_h._disabled_detours):
            _h._disabled_detours.discard(_d)
            _h._before_detours.append(_d)
            print("re-enabled", getattr(_d, "__name__", _d))
```

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| Nothing listening on 6770 | Connection refused: the game is not injected (or exited). Attach with SPEC-0040. |
| Snippet raises | Traceback comes back on the socket; game state is unchanged up to the failing line. |
| Snippet hangs (e.g. infinite loop) | The executor runs on pyMHF's asyncio loop; further messages queue. Do not loop in snippets. |
| Patched method raises in game | pyMHF disables that detour again (REQ-005 log line); the game survives. |
| Patch reads bad memory | Game may freeze then crash. Not recoverable from the executor (REQ-007). |
| Host side detached | Executor keeps working: it lives in the game process. |

## Verification

1. Hook census: send
   `from pymhf.core.hooking import hook_manager as _hm` then, for each `_k, _h` in `_hm.hooks.items()`,
   print `_k.name, _h.state, len(_h._before_detours), len(_h._after_detours), len(_h._disabled_detours)`.
   Every mod hook should be listed with 0 disabled.
2. Liveness: print `mod_manager.mods["LivePosition"]._next` and `time.time()`; they should be within
   0.25 s of each other while the player is in game.
3. Re-enable drill: after the `TryAddMarker` incident, the census showed `on_add_marker` under
   disabled; after the patch it was back under before-detours and `visor-*.jsonl` resumed
   `marker_add` lines.
4. Live load: `load_single_mod` for `galaxy_log.py` returned a non-zero hook count and
   `systems-YYYYMMDD.jsonl` appeared within seconds.

## Lessons

See [SPEC-0049](../lessons/SPEC-0049-live-hook-lessons.md): the app-object hot-patch crash (REQ-007)
and the enum decode that raised and got the marker detour disabled (REQ-005, REQ-008).
