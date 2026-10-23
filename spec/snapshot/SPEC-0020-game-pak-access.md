---
id: SPEC-0020
title: 'Game pak access: find, unpack, convert, grep and cite packed files'
description: 'A command-line tool over the installed game''s .pak archives: an index of every packed path, regex-driven unpacking, MBIN to MXML conversion, content search and citable excerpts, all cached by Steam build id.'
type: snapshot
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: []
supersedes: []
superseded_by: []
related_specs: [SPEC-0021, SPEC-0023, SPEC-0024, SPEC-0025, SPEC-0026, SPEC-0030, SPEC-0039]
screenshots: []
---

# SPEC-0020: Game pak access

## Summary

No Man's Sky ships its data in 97 `.pak` archives (plus `BankSignatures.bin`) holding about 195,000 packed paths
(build 25800958). Almost every question this project asks of the game — a recipe time, a hotspot
strength, the radius of Harvest All, the collision box of a generator — is answered by one field
in one of those files, and the hard part is knowing *which* file. This spec defines `nms-pak`, a
small standard-library CLI that indexes every packed path once per game build, unpacks files
chosen by regular expression, converts them from the game's binary MBIN format to readable MXML,
searches what has been unpacked, and stores line-range excerpts as research sources with
provenance. Every other tool in the area (icons, glyphs, footprints, the Harvest All mod, the
compendium) unpacks through it; the table extractor (SPEC-0021) drives the same two community
tools directly.

## Inputs and outputs

**Environment it needs** (all generic; nothing from the original's machines):

| Thing | Default | Override |
|---|---|---|
| Steam library `steamapps` dir | `~/.local/share/Steam/steamapps` | — |
| Game pak directory | `<steamapps>/common/No Man's Sky/GAMEDATA/PCBANKS` | `NMS_PCBANKS` |
| Tools directory holding `hgpaktool`, `MBINCompiler-linux`, `dotnet8/` | a tools directory chosen at install | `NMS_TOOLS` |
| Work/cache directory | `<tmp>/nms-pak/<build id>` | `NMS_PAK_WORK` |
| Repository root (for `cite`) | the directory above the script | — |

**Third-party tools** (both by monkeyman192, single downloadable binaries, Linux builds exist):

- **HGPAKtool** — reads and unpacks `.pak` files from game version 5.50 (Worlds Part II) on.
  Used flags: `-L` (list contents of the given paks), `-U` (unpack), `-O DIR` (output dir),
  `-f PATTERN` (repeatable filter; glob or exact upper-case path).
- **MBINCompiler** — converts MBIN <-> MXML. Used as
  `MBINCompiler-linux convert -y --force -q FILE...`. The Linux build needs a .NET 8 runtime;
  point `DOTNET_ROOT` at it if it is not system-wide. There is one MBINCompiler release per game
  version; an old release mis-reads a new build's MBINs. MBINCompiler writes a log file beside its
  own binary, so the tools directory must be writable by the user who runs it.

**Outputs:**

- `<work>/index.json` — `{ "<packed path>": "<pak file name>" }` for every path in every pak.
- `<work>/files/<packed path>` — unpacked files, and for each `x.mbin` an `x.MXML` beside it.
- `research/sources/game-data/<build>/<relative path>.L<a>-<b>.xml` — cited excerpts (SPEC-0026).
- stdout: paths and matches; stderr: counts and hints.

**Build id:** read from `<steamapps>/appmanifest_275850.acf` with the regex
`"buildid"\s+"(\d+)"`; `unknown` if the file or the field is missing.

## Requirements

### SPEC-0020-REQ-001: Build-keyed cache

The tool shall place its index and every unpacked file under a work directory keyed by the
installed Steam build id, so that a game update starts with an empty cache and never mixes files
from two builds.

### SPEC-0020-REQ-002: Path index

When any command needs the index and `<work>/index.json` does not exist, the tool shall run
`hgpaktool -L <every *.pak in PCBANKS, sorted>` **from a fresh temporary directory as the current
working directory**, read the `filenames.json` that hgpaktool writes there
(`{ "<pak path>": ["<packed path>", ...] }`), invert it to `{ packed path: basename(pak) }`, write
that as `index.json`, and delete the temporary directory. (hgpaktool `-L` writes `filenames.json`
into the current directory whatever `-O` says; see SPEC-0039.) Building the index takes about a
second; it is reused for the life of the build.

### SPEC-0020-REQ-003: `find REGEX`

The tool shall print every indexed packed path matching the case-insensitive regular expression
as `<pak name padded to 32> <packed path>`, sorted by path, and print
`<n> match(es) in build <id>` to stderr. `find` never unpacks anything.

### SPEC-0020-REQ-004: `get REGEX [--all]`

When `get` is run, the tool shall:

1. Select indexed paths matching the case-insensitive regex. If none, exit non-zero with
   `no packed path matches`.
2. If more than **200** match and `--all` was not given, exit non-zero naming the count
   (protects against unpacking half the game by accident).
3. Group matches by pak; for each pak run
   `hgpaktool -U -O <work>/files -f <PATH UPPER CASE> [-f ...] <pak>`.
4. Collect the unpacked `.mbin` files that have no `.MXML` beside them yet and convert them with
   MBINCompiler in batches of **100** paths per invocation, with `DOTNET_ROOT` defaulted to
   `<tools>/dotnet8`.
5. For each match print the absolute `.MXML` path (packed path with `.mbin` replaced by `.MXML`)
   if it exists, else the raw unpacked path for non-MBIN files (`.dds`, `.txt`, ...), else
   `(not converted) <path>`.

The printed list is the interface other tools consume: they pass an anchored regex, read stdout,
and pick lines by suffix.

### SPEC-0020-REQ-005: Idempotent get

While an `.MXML` already exists for a packed `.mbin`, the tool shall not re-convert it. Re-running
`get` with the same regex shall be cheap and print the same paths.

### SPEC-0020-REQ-006: `grep PATTERN [--in REGEX]`

The tool shall search, case-insensitively, every line of every `.MXML` under `<work>/files`
(optionally only files whose path matches `--in`), print `<file>:<line>: <stripped line>` per hit,
and print to stderr the hit count plus the reminder that only unpacked files are searched.
Content search over the whole game is deliberately not offered: `find` the likely files by path,
`get` them, then `grep`.

### SPEC-0020-REQ-007: `show PATH [--lines A-B]`

The tool shall print an unpacked file with right-aligned 6-wide line numbers. `PATH` may be an
absolute path or a packed path (`.mbin` is mapped to its `.MXML`). `--lines A` means one line;
`A-B` an inclusive range.

If `PATH` has not been unpacked, then the tool shall exit with `not unpacked: <path> (get it first)`.

### SPEC-0020-REQ-008: `cite PATH --lines A-B --why TEXT`

The tool shall copy the line range of an unpacked MXML to
`research/sources/game-data/<build>/<path relative to work/files>.L<A>-<B>.xml`, prefixed by an
XML comment header with exactly these keys:

```
<!--
  source: game-data
  build: <build id>
  pak: <pak name from the index, or ?>
  path: <packed path, .mbin>
  lines: <A>-<B> of the MBINCompiler MXML
  retrieved: <YYYY-MM-DD>
  why: <TEXT>
-->
```

and print the repository-relative destination.

If the destination already exists, then the tool shall refuse and exit non-zero: a cited excerpt
is a record of what one build said and is never overwritten.

### SPEC-0020-REQ-009: Standard library only

The tool shall use only the language's standard library and the two external binaries; no
package manager install.

### SPEC-0020-REQ-010: Packed-path conventions

The tool shall treat packed paths as lower case with forward slashes (as `-L` lists them), pass
them to hgpaktool's `-f` upper-cased, and expect MBINCompiler's output extension to be upper-case
`.MXML`. Consumers that read paths out of game files (scene references are written
`MODELS\PLANETS\...\X.SCENE.MBIN`) shall lower-case them and replace `\` with `/` before matching
against the index.

## Degraded behaviour

- **Game not installed / PCBANKS missing:** index building fails with the OS error; nothing is
  cached. Every dependent tool fails loudly, which is correct: there is no other source.
- **No Steam manifest:** build id becomes `unknown`; the cache key is `<tmp>/nms-pak/unknown` and
  `cite` writes under `.../game-data/unknown/`. Usable, but the staleness checks in SPEC-0026 can
  no longer tell builds apart; fix the manifest path rather than living with it.
- **MBINCompiler missing, wrong version or no .NET runtime:** conversion silently yields no MXML;
  `get` prints `(not converted) <path>` for each MBIN so callers see it. Callers (SPEC-0021,
  SPEC-0024, SPEC-0025) treat a missing MXML as fatal for the file they needed.
- **hgpaktool missing or failing:** `subprocess` raises; the command dies with the error.
- **Cache from a previous build:** never read, because the key changed. Old work directories can
  be deleted at will.
- **Over-broad regex:** refused above 200 matches without `--all`.

## Verification

1. `nms-pak find 'regionhotspotstable'` prints one line:
   `NMSARC.Precache.pak  metadata/simulation/scanning/regionhotspotstable.mbin`, and stderr
   says `1 match(es) in build <id>`.
2. `nms-pak find '.'` (count only) reports about 195,000 paths across 97 paks on build 25800958;
   a later build will differ but should be the same order of magnitude.
3. `nms-pak get 'regionhotspotstable'` prints one `.MXML` path; the file starts with
   `<?xml ...?>`, then `<!--File created using MBINCompiler version (x.y.z)-->`, then
   `<Data template="cGcRegionHotspotsTable">`, and contains
   `<Property name="RegionHotspotsPoleSpacing" value="900.000000" />`.
4. `nms-pak grep 'GcHarvestPlantAction' --in plantroom` after getting the freighter plant room
   floor entity (SPEC-0025) prints one hit followed within two lines by `Radius` `6.000000`.
5. `nms-pak cite <that MXML> --lines 1-5 --why test` creates a file; running it again refuses.
6. After changing the build id in a copy of the manifest (or setting `NMS_PAK_WORK` elsewhere),
   `find` rebuilds the index rather than reusing the old one.
7. No `filenames.json` appears in the directory you ran the tool from.

## Lessons

See [SPEC-0039](../lessons/SPEC-0039-game-data-lessons.md). The two that matter most here:

- **hgpaktool `-L` drops `filenames.json` in the current directory**, ignoring `-O`. Run it from a
  temporary directory, or a stray `filenames.json` turns up as an untracked file in whatever repo
  you happened to be in (it did, in an unrelated repository).
- **Search by path first, content second.** The setting you want is often on an object you did not
  think of (the Harvest All radius is on the room's floor entity, not the crop or the globals); an
  early answer called it "hardcoded" after grepping only the globals and the crop.
