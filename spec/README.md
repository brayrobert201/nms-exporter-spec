# spec

What the system does (`snapshot/`), why it is the way it is (`decisions/`), and what it cost to
learn (`lessons/`). The audience is someone rebuilding this with an AI coding agent and none of
the original's infrastructure: every spec gives the formats, constants and algorithms needed to
reproduce the behaviour, and names what it needs from the environment generically (a Postgres, a
secret store, a reverse proxy) rather than as any particular product.

## Layout

```
spec/
  README.md        this file
  ROADMAP.md       every spec, by area, with status
  templates/       snapshot, decision, lesson
  snapshot/        current behaviour as EARS requirements
  decisions/       architecture decision records
  lessons/         gotchas per area, ordered by time saved
```

## Areas

| area | covers | SPEC range |
|---|---|---|
| `save-exporter` | reading the save, metrics, farms/refiners/planner, Grafana, wall screens, save backup | 0001–0019 |
| `game-data` | unpacking game files, tables, icons, glyphs, part footprints, a loose-file mod, the mechanics compendium method | 0020–0039 |
| `live-hook` | in-process hooking of the running game under Proton, the mods, hot-patching, frames, storms, hotspots | 0040–0059 |
| `map-ui` | the browser map and systems pages | 0060–0079 |
| `storage-deploy` | Postgres store, environment, service, machine identity for secrets, exposure | 0080–0099 |

## Numbering

One global sequence, `SPEC-NNNN-slug.md`, shared by all three directories. Ranges above are
reserved per area so areas can be written in parallel; numbers are never reused.

## Frontmatter

Every file starts with YAML frontmatter per its template in `templates/`. `related_specs` links
across areas; `screenshots` lists files under `../screenshots/`.
