---
id: SPEC-0091
title: 'A locked uv environment for the one dependency'
description: 'Why the exporter relaxed its standard-library-only rule for psycopg, and why that dependency comes from a uv project with a committed lock built against the system Python.'
type: decision
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0082, SPEC-0090, SPEC-0012]
---

# SPEC-0091: A locked uv environment for the one dependency

## Context

The exporter was written standard library only, with "nothing to install" as a stated
feature: it runs unattended for weeks on a desktop, and every dependency is something that can
break on an OS upgrade. It decodes LZ4 save chunks, signs S3 requests with SigV4 and serves HTTP
without a single package.

Postgres (SPEC-0090) broke that. Python's standard library has no Postgres client. The options
were to implement the wire protocol (startup, SCRAM-SHA-256 authentication, TLS upgrade,
extended query protocol, type parsing) or take a driver. Implementing the protocol is possible
but is a security-sensitive component maintained by one person for one program.

Given a driver, it has to come from somewhere: the OS package (`python3-psycopg`), `pip install
--user`, a hand-made venv, or a project manager with a lock.

## Decision

1. Relax "standard library only" **for this one dependency and only in the exporter process**:
   `psycopg[binary]>=3.2,<4`. Everything else stays standard library, including the secret-sync
   script, which runs on the system Python so it works before any environment exists.
2. Declare it in `pyproject.toml` (`[tool.uv] package = false`) with a committed `uv.lock`.
3. Install with `uv sync --frozen --no-dev --python /usr/bin/python3` and
   `UV_PYTHON_DOWNLOADS=never`: exactly the locked versions, on the distribution's Python, no
   uv-managed interpreter.
4. The service runs `/opt/nms-exporter/.venv/bin/python`.
5. The `binary` extra, so libpq is bundled in the wheel and nothing needs installing from the OS
   package manager.

The owner chose uv over staying standard-library-only for this project when the store moved to
Postgres; the rule still applies to other projects of the same family.

## Alternatives considered

- **Write a minimal Postgres client.** No dependency, but SCRAM and TLS by hand, for one
  program. Rejected.
- **OS package (`python3-psycopg`).** No new tooling, security updates from the distribution.
  Rejected: the version follows the distribution's release, not the project; nothing pins it,
  so behaviour can change under an upgrade with no change in the repo; and it is not available
  under that name everywhere a rebuilder might run.
- **`pip install` into the system or user site.** Unpinned unless a requirements file is kept by
  hand, conflicts with the OS's own packages, and modern distributions refuse it (PEP 668).
- **A venv made by hand with `pip install -r requirements.txt`.** Works, but a requirements file
  without hashes is not a lock, and recreating it reproducibly is the job uv does in one command.
- **uv with its own Python.** Rejected via `--python /usr/bin/python3` and downloads disabled:
  the distribution's Python receives security updates with the rest of the OS; a downloaded
  interpreter would not.

## Consequences

- Install needs `uv` on the machine (an OS package or its standalone binary). The install script
  checks and stops early if it is missing.
- A fresh install needs the package index once; later installs can use the cache.
- The environment is reproducible from the lock, and the lock is reviewed like code.
- A minor upgrade of the system Python breaks the venv; rerunning install rebuilds it. A
  rebuilder should treat "rerun install after an OS upgrade" as routine.
- Development uses the same `.venv` (the dev-deploy script runs the working tree with
  `.venv/bin/python`).
- The rule "no dependencies" becomes "one dependency, locked". The bar for a second one is the
  same as for the first: no reasonable standard-library way to do it.

Revisit if Python's standard library ever ships a Postgres client (unlikely), if the store moves to
SQLite (then the environment is unnecessary and the project can return to "nothing to install"),
or if a second dependency is proposed (consider whether it is worth it at all before adding it to
the lock).
