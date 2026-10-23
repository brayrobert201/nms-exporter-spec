---
id: SPEC-0001
title: 'Finding and decoding the save'
description: 'Locate the newest save*.hg under the Proton prefix, decode its LZ4 chunk chain, and translate the obfuscated JSON keys with the MBINCompiler mapping.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: '2026-10-23'
status: current
depends_on: []
supersedes: []
superseded_by: []
related_specs: [SPEC-0002, SPEC-0003, SPEC-0010, SPEC-0011, SPEC-0019, SPEC-0021]
screenshots: []
---

# SPEC-0001: Finding and decoding the save

## Summary

Everything in the save-exporter area starts from one file: the newest No Man's Sky save the game
wrote. On Linux the game runs under Steam's Proton, so the save lives inside the Proton prefix of
app 275850. The file is a chain of LZ4-compressed chunks wrapping one JSON document whose keys are
obfuscated three-character codes. This spec says how to find the file, when it is safe to read,
how to decompress it with nothing but the standard library, and how to turn the keys back into
readable names. It also fixes the one encoding quirk that bites every reader: wallet values are
unsigned 32-bit numbers written as signed JSON integers.

## Inputs and outputs

| Input | Where | Notes |
|---|---|---|
| Save directory | `~/.local/share/Steam/steamapps/compatdata/275850/pfx/drive_c/users/steamuser/AppData/Roaming/HelloGames/NMS/st_<steamid>` | Override with `NMS_SAVE_DIR`. |
| Save slots | `save.hg`, `save2.hg` ... `saveN.hg` in that directory | Each has a sibling manifest `mf_<name>.hg`. |
| Account file | `accountdata.hg` | Not decoded by the exporter; backed up (SPEC-0010). |
| Key mapping | `mapping.json` from the MBINCompiler GitHub release | `{"libMBIN_version": "...", "Mapping": [{"Key": ">Qh", "Value": "X"}, ...]}` |

Output: a Python `dict` (decoded JSON with readable keys), handed to the snapshot builder
(SPEC-0002).

### Chunk format

```
repeat until end of file:
  u32 magic   = 0xFEEDA1E5      (little-endian)
  u32 comp    = compressed size of the block that follows
  u32 uncomp  = uncompressed size
  u32 zero    = 0
  comp bytes  = one raw LZ4 *block* (not an LZ4 frame: no frame header, no checksums)
```

Concatenate the decompressed blocks, strip trailing NUL bytes, decode as UTF-8 with
`errors="replace"`, then parse JSON.

### LZ4 block decoding (the whole algorithm)

```
while src not exhausted:
  token = next byte
  lit = token >> 4;  if lit == 15: keep adding next bytes while byte == 255 (add the final one too)
  copy lit literal bytes to dst
  if src exhausted: stop            # last sequence has literals only
  offset = u16 little-endian
  ml = token & 0x0F; if ml == 15: extend as for lit;  ml += 4
  copy ml bytes from dst[len(dst) - offset]; if offset < ml the regions overlap -> copy byte by byte
```

A slice copy when `offset >= ml` and a byte loop otherwise is fast enough in pure Python for a
~10 MB decoded save (decode time is exported as `nms_collector_decode_duration_seconds`).

## Requirements

### SPEC-0001-REQ-001: Find the save directory

The exporter shall use `NMS_SAVE_DIR` when set; otherwise it shall search the current user's home
and then every `/home/*` for the Proton path above with glob `st_*`, and take the first match in
sorted order. If none is found, the snapshot's `errors.save` shall say
`no save files under (no save dir found)`.

### SPEC-0001-REQ-002: Choose the newest save

The exporter shall consider every `save*.hg` in the directory with size > 0 and choose the one with
the newest mtime, regardless of slot number. (The game rotates autosaves and manual saves across
slots; "newest file" is the only reliable "current state".)

### SPEC-0001-REQ-003: Settle before reading

If the newest save's mtime is less than **2 s** in the past, the exporter shall not decode it on
this request and shall keep serving the previous snapshot. The game writes in place; a read inside
that window can see a half-written chunk chain.

### SPEC-0001-REQ-004: Decode only on change

The exporter shall remember `(path, mtime)` of the last decode attempt and shall skip decoding
when both are unchanged. A request is then a stat() of the directory plus a projection (SPEC-0003).

### SPEC-0001-REQ-005: Validate chunk magic

If a chunk header's magic is not `0xFEEDA1E5`, decoding shall fail with
`bad chunk magic 0x... at offset N`.

### SPEC-0001-REQ-006: Key mapping lookup order

The exporter shall load the mapping from the first existing of: `NMS_MAPPING`, the service cache
directory's `mapping.json`, the repository's `data/nms_mapping.json`. If none exists it shall
download the release asset
`https://github.com/monkeyman192/MBINCompiler/releases/latest/download/mapping.json` into the
cache directory once and use it. The mapping is loaded once per process.

### SPEC-0001-REQ-007: De-obfuscate recursively

The exporter shall replace every dict key found in the mapping, recursively through dicts and
lists, leaving unknown keys as they are. Values are never translated.

### SPEC-0001-REQ-008: Tolerate non-UTF-8 item ids

Procedural item ids (`<type>#<seed>`) can contain raw bytes. Decoding shall replace invalid UTF-8
rather than fail, and every later consumer (metric labels, names) shall treat ids as possibly
unprintable (SPEC-0002-REQ-009).

### SPEC-0001-REQ-009: Read wallets as unsigned

`PlayerStateData.Units`, `.Nanites` and `.Specials` (quicksilver) shall be read as uint32: a
negative value `v` becomes `v + 2**32`.

The same correction shall be applied to the *currency* counters in the `^GLOBAL_STATS` block
(ids containing `MONEY`, `NANITES`, `SPECIAL` or `UNITS`) and to nothing else: standings there
(`TRA_STANDING`, `WAR_STANDING`, `EXP_STANDING`, the `*GUILD_STAND` values) are genuinely signed
and go negative (a Gek standing of −86 was shown as 4,294,967,210 for an afternoon after the
correction was applied to every counter). Before this was
done, `MONEY_EVER` exported as `-617290418` (3,677,676,878 unsigned) and the traveller screen
showed "-617.3M earned ever".

### SPEC-0001-REQ-010: Where the useful data is

After de-obfuscation the builder shall read:

| Path | Used for |
|---|---|
| `BaseContext.PlayerStateData` (`ps`) | almost everything |
| `BaseContext.GameMode` | 1 Normal, 2 Creative, 3 Survival, 4 Permadeath, 5 Ambient |
| `CommonStateData.SaveName`, `.TotalPlayTime` | save name, play time |
| `Version`, `Platform`, `ActiveContext` | `nms_save_info` labels |
| `ps.TimeStamp` | the game's own Unix time of the save |
| `ps.SaveSummary` | the "Within X" line the load menu shows |
| `ps.DifficultyState.Preset.DifficultyPresetType` | difficulty label |
| `DiscoveryManagerData.DiscoveryData-v1.Store.Record` | discovery count |

Item ids in the save carry a leading `^` (`^FUEL1`); strip it before every table lookup.

### SPEC-0001-REQ-011: Never raise into the server

If decoding or building fails, the exporter shall increment `decode_failures`, set
`errors.save = "<file>: <exception>"`, record the `(path, mtime)` as attempted so a broken file is
not re-decoded on every scrape, and keep serving the last good snapshot.

### SPEC-0001-REQ-012: Decode tool

Where a human wants to inspect a save, a script shall dump any `.hg` to readable JSON
(`nms-decode-save.py SAVE.hg out.json`) using the same decoder and mapping.

## Degraded behaviour

| Missing / failing | Result |
|---|---|
| No Proton prefix, no `st_*` directory | `errors.save` explains; `/metrics` serves `nms_collector_up 0` and the collector/backup counters only; `/json` is `{"errors": {...}, "backup": {...}}`. |
| Directory exists, no non-empty `save*.hg` | Same, message names the directory. |
| Newest save < 2 s old | Previous snapshot served unchanged; picked up on the next request. |
| Corrupt or truncated save | `errors.save` set, failure counter +1, last good snapshot kept, file not retried until its mtime changes. |
| No mapping and no network | The download raises inside the decode `try`; `errors.save` carries the URL error. Ship `data/nms_mapping.json` in the repo so this never happens on a fresh install. |
| Mapping out of date after a game update | Unknown keys stay obfuscated; fields read as missing and sections come out empty rather than wrong. Refresh the mapping. |

## Verification

1. `scripts/nms-decode-save.py <newest save> /tmp/s.json` and check that
   `BaseContext.PlayerStateData.Units` exists (readable keys) and that `SaveSummary` matches the
   in-game load menu.
2. Round-trip the LZ4 decoder: compress random data with any LZ4 *block* encoder, decode, compare.
   Include a payload with long runs (overlapping matches) and one longer than 15 + 255 literals.
3. `curl -s localhost:9877/metrics | grep nms_collector` - `nms_collector_up 1`, decodes > 0,
   failures 0, decode duration a few seconds at most.
4. Units above 2,147,483,647 must export positive: compare `nms_units` with the in-game balance.
5. Save in game; within one scrape after 2 s, `nms_save_mtime_seconds` advances and
   `nms_collector_decodes_total` increases by one.

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md). Top one: the wallet is uint32
written signed - a balance of 2,178,378,504 units showed as -2,116,588,792 until the reader added
2^32, and lifetime money counters still need the same fix.
