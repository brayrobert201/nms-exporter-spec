---
id: SPEC-0026
title: 'Mechanics compendium: questions, stored sources, evidence levels, concepts'
description: 'The method and file formats for a sourced compendium of how the game works: a question register, a raw source store, an evidence ladder, one-concept-per-file knowledge with trust entries, a lint, and a change log.'
type: snapshot
area: game-data
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0020]
supersedes: []
superseded_by: []
related_specs: [SPEC-0022, SPEC-0025, SPEC-0030, SPEC-0039]
screenshots: []
---

# SPEC-0026: Mechanics compendium

## Summary

Questions about how the game works come up constantly in play ("does the extractor slow when I pipe
more in?", "how far does Harvest All reach?"). Answered ad hoc they get answered wrongly, then again,
differently. The compendium makes each answer a file that names its evidence, the build it was
observed on, and who has checked it, so it can be looked up once and reused by the player, by the
exporter's code, and by whichever coding-agent session is asked next. It is a method plus a few
standard-library tools, not a service. It lives in the repository as two trees: `research/`
(questions, raw sources, runs) and `knowledge/` (concepts).

## Inputs and outputs

```
research/
  index.md                       charter: purpose, layout, evidence ladder, review lenses
  questions/<slug>.md            one open question each (type: Question)
  questions/index.md             generated
  sources/game-data/<build>/...  excerpts stored by `nms-pak cite` (SPEC-0020)
  sources/web/<date>-<slug>.md   pages stored by `research-fetch`
  sources/measurements/<date>-<slug>.md   hand-written: method, readings, build, date
  runs/<YYYY-MM-DD-slug>/        brief.md, args.json, plan.md, raw/, review-*.md,
                                 synthesis.md, promoted.md
knowledge/
  meta/conventions.md            the contract below
  mechanics/<slug>.md            type: Mechanic      how something works
  data-locations/<slug>.md       type: DataLocation  where a setting lives, how to change it
  tools/<slug>.md                type: Tool          tools and data sources, what each is good for
  index.md                       generated
  log.md                         newest-first change log
```

Tools (all standard library): `nms-pak cite`, `research-fetch URL --slug S --why W [--raw]`,
`research-index` (regenerates both index files from frontmatter), `okf-lint` (checks them).

## Requirements

### SPEC-0026-REQ-001: Questions come from decisions

Every question file shall carry frontmatter `type: Question`, `title` (the question),
`status` (`open | partial | answered | dropped`), `asked` (date), `decides` (the play decision,
exporter number or mod value the answer changes), optional `tags`, `runs`, `answered_by` (concept
links). The body has **Why it matters**, **Known so far**, **What would answer it** and, where a
measurement is needed, a **Measurement plan** precise enough for the player to execute (which
hotspot, how many extractors, how long, when to save, what to read back).

A finding is most valuable when it changes a play decision, the exporter or planner, or a mod;
reference facts that change none of these are kept but are not what research is for.

### SPEC-0026-REQ-002: If a claim cites it, it is stored

Every source a concept cites shall exist in the repository:

- **game-data:** `nms-pak cite` excerpt (header: source, build, pak, path, lines, retrieved, why).
  Never overwritten.
- **web:** `research-fetch` stores the page text verbatim (scripts, styles and markup stripped,
  nothing else changed) under a header with URL, final URL, retrieval date, HTTP status,
  content type, sha256 of the body and why. Never overwritten. For MediaWiki pages fetch the raw
  form (`?action=raw`).
- **measurements:** a Markdown file with frontmatter `type: Measurement`, `title`, `build`, `date`,
  `method`, then the raw readings (tables), not just conclusions.
- `data/nms_tables.json#<section>` is acceptable for values the table extractor pulled from the
  paks; its `meta` gives the build.

A run's searcher returns are kept under the run's `raw/` with a header saying who produced them,
how, and that they are unreviewed.

### SPEC-0026-REQ-003: Evidence ladder

Each concept shall carry one `evidence_level`, strongest first:

1. `game-data` — the installed build's own files say so (excerpt cited).
2. `code` — the executable does so: a runtime hook or disassembly, method recorded.
3. `measured` — measured in game or across saves, method and readings stored.
4. `official` — Hello Games patch notes or statements.
5. `community-measured` — someone else measured it and published the method.
6. `community-claim` — wiki, forum, video or mod page without a method.
7. `none`.

The reviewer, never the finder, assigns the level. **A game-data value is not yet a behaviour**:
"Radius is 6.0" is game-data; "Harvest All reaches 6 m around the room" needs code or a
measurement too. Concepts keep the two apart, typically as "Evidence" vs "Scope and caveats".

### SPEC-0026-REQ-004: Concept frontmatter

Each concept shall start with YAML frontmatter:

| Field | Required | Meaning |
|---|---|---|
| `type` | yes | `Mechanic`, `DataLocation`, `Tool` |
| `title` | yes | |
| `status` | yes | `draft`, `stable` or `superseded` |
| `evidence_level` | yes | REQ-003 |
| `builds` | unless level is `community-claim`, `official` or `none` | Steam build ids observed on |
| `stale_after` | yes | `next-build`, or a date for dated sources |
| `sources` | yes | repo paths, each must exist |
| `generated` | yes | `{by: <role>, at: <date>}` |
| `verified` | yes | list of `{by: <role>, at, run?}`, may be `[]` |
| `tags`, `related` | no | |
| `verdict` | no | one quotable paragraph |

Frontmatter is kept to scalars, `[inline lists]` and `- ` lists so a small reader, not a YAML
library, can parse it.

### SPEC-0026-REQ-005: Trust entries name roles

`generated` and `verified` entries shall name a role, never a model: `machine:session` (written
directly in a working session, unreviewed), `machine:searcher` (found by a research-run searcher),
`machine:reviewer` (survived an adversarial review in run `<run>`), `human:<owner>` (the owner has
read it and agrees — **only the owner writes this**). A concept with only machine session/searcher
entries is unreviewed and says so with `verified: []`.

### SPEC-0026-REQ-006: Concept body

The body shall give, in order: **the answer** in one or two sentences with the number if there is
one (bold); **Evidence** (each source and what it shows; game-data values with file and field);
**Scope and caveats** (build, mode, what is inferred); **Open** (unknowns, linking question files).
Live observations and measurements may have their own dated sections.

### SPEC-0026-REQ-007: Review lenses

A reviewer shall check every claim for: **version laundering** (a pre-update wiki line cited as
current; every claim names its build or the page's date/patch); **data is not behaviour** (a table
field is not proof the code uses it; the trading cost table exists and is unused);
**mode and platform** (difficulty presets, custom settings, expeditions, permadeath, consoles;
default scope is PC, Normal preset, the main save); **mod pages as evidence** (a mod page says what
its author intended, for their version); **absence in the data** ("not in the paks" needs the
stored `find`/`grep` census that would have found it; until then "not found in the files we
unpacked", and "hardcoded" is a hypothesis).

### SPEC-0026-REQ-008: Lint

`okf-lint` shall walk `knowledge/` (skipping `meta/`) and `research/questions/`, skip `index.md`
and `log.md`, and exit 1 on any error: no frontmatter; a missing required field (REQ-004 for
concepts; `type, title, status, asked, decides` for questions); a status outside the allowed set;
an unknown evidence level; a concept whose level is not `community-claim`, `official` or `none`
but has no `builds`; a concept with no `sources`; a `sources`, `related` or `answered_by` path that
resolves neither from the repository root nor from the file's directory (a `#fragment` is ignored).
It shall **warn** (not fail) when a `stale_after: next-build` concept's `builds` does not include
the installed build (read from the Steam manifest), when a dated `stale_after` is past, and when a
file carries a `human:<owner>` entry (only the owner adds that; an environment variable silences
the warning for the owner's own runs). It ends with `<n> files, <e> error(s), <w> warning(s)`.
Stale concepts are re-checked and the new build appended, never deleted.

### SPEC-0026-REQ-009: Generated indexes

`research-index` shall regenerate `research/questions/index.md` (table: status, question link,
decides; open before partial before answered) and `knowledge/index.md` (one table per directory:
concept link, evidence, builds, status) from frontmatter, each headed by a "generated; edit the
files it lists" comment.

### SPEC-0026-REQ-010: Log

Every change to the bundle shall add an entry at the top of `knowledge/log.md`:
`## YYYY-MM-DD - <short title>` and bullets saying what landed and from where (source paths).

### SPEC-0026-REQ-011: Small questions vs runs

A single-gap question with one or two sources shall be answered in-session: store sources, write
or update the concept (`generated: machine:session`, `verified: []`), update the question,
regenerate indexes, lint, log. A question needing fan-out or adversarial review gets a **run**:
a brief agreed with the owner first (question, build and mode scope, what is known, which decision
it changes, depth: orientation 1–2 sub-questions, standard 3–4, deep 5–6), then plan, searchers,
reviews, synthesis, and promotion into `knowledge/` done by the orchestrating session, never inside
the run.

### SPEC-0026-REQ-012: Privacy

Saves hold player and base names and galactic addresses. They shall never appear in a web query or
a stored source; describe situations generically. (For a public rebuild: also no hostnames,
usernames or account identifiers.)

## Degraded behaviour

- **Game updated:** every `next-build` concept warns; nothing is deleted. Re-check by re-citing
  (the new build gets its own `sources/game-data/<build>/` directory) and appending to `builds`.
- **Web source gone:** the stored copy stands; the concept still resolves.
- **Fetch refused** (e.g. a wiki answering HTTP 402 to automated clients): record that in the
  run's raw notes; do not paraphrase from memory as if fetched.
- **No reviewer available:** concepts stay `draft` with `verified: []`; consumers treat them as
  unreviewed.

## Verification

- `okf-lint` exits 0 on a fresh checkout; delete one cited source file and it exits 1 naming it.
- Change the installed build id: every `next-build` concept appears as a warning.
- `research-index` is idempotent: run twice, no diff.
- Pick any number in `knowledge/mechanics/` and follow its `sources` to a stored excerpt showing
  that number with its build, pak, path and line range.
- Seed set on the original (build 25800958): six mechanics (base power, extraction hotspots,
  Harvest All radius, save coordinates, units wallet, weather and storms), one data location
  (Harvest All radius), one tool survey; ten questions (five open, five partial).

## Lessons

See [SPEC-0039](../lessons/SPEC-0039-game-data-lessons.md). The two that justify the method:

- **"Hardcoded" was said too early.** The Harvest All radius was declared hardcoded after checking
  two files; it is a data value on the room entity. Absence needs a stored census.
- **The obvious reading of a table was wrong.** Storm timing by StormFrequency predicted 120–180 s
  storms; the measured storm on an Extreme planet lasted 315 s. Game-data and measured evidence are
  different levels for a reason.
