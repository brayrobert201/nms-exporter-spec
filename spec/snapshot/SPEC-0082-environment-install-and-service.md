---
id: SPEC-0082
title: 'Python environment, install and the systemd service'
description: 'A locked uv environment for the one dependency (psycopg), an install script that builds it offline against the system Python, and a per-Steam-user systemd unit with optional env files.'
type: snapshot
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0080]
supersedes: []
superseded_by: []
related_specs: [SPEC-0091, SPEC-0084, SPEC-0099, SPEC-0011, SPEC-0010, SPEC-0012]
screenshots: []
---

# SPEC-0082: Python environment, install and the systemd service

## Summary

The exporter is one Python program that must run on the gaming PC, as the Steam user, because it
reads that user's Proton save directory and the live hook's output inside the Proton prefix. It
is installed to a fixed directory and run by a systemd template unit instantiated with the
Steam user's name. Everything on the save side is Python standard library; the Postgres store
(SPEC-0080) needs one third-party package, `psycopg` (version 3, the `binary` extra so libpq is
bundled). That dependency lives in a uv project with a committed lock file, and the install
builds the environment from the lock, offline from Python's point of view (no interpreter
downloads), against the system Python. Secrets reach the service through optional root-only
environment files that a separate timer writes (SPEC-0084).

## Inputs and outputs

| Item | Path / value |
|---|---|
| install root | `/opt/nms-exporter` (program, package, data, wall-screen configs, `pyproject.toml`, `uv.lock`, `.venv/`) |
| interpreter | `/opt/nms-exporter/.venv/bin/python` (a venv over `/usr/bin/python3`) |
| unit | `/etc/systemd/system/nms-exporter@.service`, instance = Steam user |
| env files | `/etc/nms-exporter/s3.env` (save backup, SPEC 0001–0019), `/etc/nms-exporter/db.env` (this area), both optional, root, mode 600 |
| secret-store credentials | `/etc/nms-exporter/bao/role_id`, `/etc/nms-exporter/bao/secret_id`, root, mode 600 (SPEC-0084) |
| state | `/var/lib/nms-exporter` (StateDirectory: fill-level history), `/var/cache/nms-exporter` (CacheDirectory) |
| uv cache for the install | `/var/cache/nms-exporter-uv` |
| listen | `0.0.0.0:9877` (`NMS_EXPORTER_HOST`, `NMS_EXPORTER_PORT`) |

### `pyproject.toml` (normative)

```toml
[project]
name = "nms-exporter"
version = "0.1.0"
description = "No Man's Sky save exporter, wall screens and live map"
requires-python = ">=3.12"
# psycopg for the history store; everything else is the standard library.
# binary: bundles libpq, so nothing needs installing from the OS package manager.
dependencies = ["psycopg[binary]>=3.2,<4"]

[tool.uv]
package = false
```

`package = false`: the project is an application, not an installable package; uv only builds the
environment. The lock resolved to psycopg 3.3.6 / psycopg-binary 3.3.6 (plus typing-extensions
and tzdata) at the time of writing; the lock, not this number, is authoritative.

### `db.env` (normative variable names)

```
# written by the secret sync; do not edit by hand
PGHOST=<database load-balancer address>
PGPORT=5432
PGDATABASE=nms
PGUSER=<db_username from the secret store>
PGPASSWORD=<db_password from the secret store>
PGSSLMODE=require
```

These are libpq's own variables, so the program passes no connection string at all.
`PGSSLMODE=require` encrypts but does not verify the server certificate (SPEC-0083 explains why
that is acceptable on the LAN). To verify instead, add `PGSSLROOTCERT=<path to the operator's
CA>` and use `verify-ca`.

## Requirements

### SPEC-0082-REQ-001: Locked environment

The repository shall contain `pyproject.toml` (above) and a committed `uv.lock`. Adding or
upgrading a dependency shall be done with `uv add` / `uv lock` and committed together.

### SPEC-0082-REQ-002: Install builds the environment from the lock only

The install script shall run, as root, after copying the program into the install root:

```bash
UV_PYTHON_DOWNLOADS=never UV_CACHE_DIR=/var/cache/nms-exporter-uv \
  uv sync --frozen --no-dev --python /usr/bin/python3 --project /opt/nms-exporter --quiet
```

`--frozen` refuses to touch the lock (install fails rather than resolving something new);
`--no-dev` skips development groups; `--python /usr/bin/python3` and
`UV_PYTHON_DOWNLOADS=never` forbid uv from fetching its own interpreter, so the service runs on
the distribution's patched Python. Packages are still fetched from the index on first install
(or from the cache).

### SPEC-0082-REQ-003: uv must be present

If `uv` is not on `PATH`, the install script shall stop with "uv not found" and a hint to install
it from the OS package manager, before changing any unit.

### SPEC-0082-REQ-004: Instance per Steam user

The unit shall be a template, `nms-exporter@.service`, with `User=%i`, so
`nms-exporter@<steam user>` reads that user's save. The install script shall take the user from
`NMS_USER`, else `SUDO_USER`, and refuse to run without one or without root.

### SPEC-0082-REQ-005: Unit shape

The unit shall contain at least:

```ini
[Unit]
Description=No Man's Sky save exporter (%i's saves)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=%i
WorkingDirectory=/opt/nms-exporter
ExecStart=/opt/nms-exporter/.venv/bin/python /opt/nms-exporter/nms_exporter.py
Environment=NMS_EXPORTER_HOST=0.0.0.0
Environment=NMS_EXPORTER_PORT=9877
Environment=PYTHONUNBUFFERED=1
EnvironmentFile=-/etc/nms-exporter/s3.env
EnvironmentFile=-/etc/nms-exporter/db.env
StateDirectory=nms-exporter
CacheDirectory=nms-exporter
Restart=on-failure
RestartSec=5s
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_INET AF_INET6
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

The leading `-` on each `EnvironmentFile` makes it optional: absent, the unit still starts.
Without `db.env` the store is unconfigured and SPEC-0081 applies. `RestrictAddressFamilies`
excludes `AF_UNIX`, which is one more reason the "not configured" check must precede any libpq
call (a Unix-socket attempt would fail with a permissions-flavoured error).
`ProtectHome=read-only` is enough: the program only reads under the user's home.

### SPEC-0082-REQ-006: Env files written by root only

`/etc/nms-exporter` shall be created mode 700; env files inside it mode 600, owned by root. The
service reads them through systemd (as root, before dropping to `%i`), so the Steam user never
needs read access.

### SPEC-0082-REQ-007: Secret-sync timer enabled only when credentialed

The install script shall install the secret-sync service and timer (SPEC-0084) every time, and
enable the timer only if both `/etc/nms-exporter/bao/role_id` and `.../secret_id` exist and are
non-empty.

### SPEC-0082-REQ-008: Restart on install

The install script shall `systemctl daemon-reload`, `enable --now` and then `restart` the
instance, so a reinstall always runs the new code.

### SPEC-0082-REQ-009: Test instance beside the live one

The development deploy script shall, before installing, run the working tree's program with the
working tree's `.venv/bin/python` on port 9878 (`NMS_EXPORTER_PORT=9878`, a scratch
`NMS_STATE`), wait up to 20 s for `/healthz`, fetch `/json`, fail if its `errors` map is not
empty, count `nms_` metrics, and then stop the test instance. It shall find the test instance
**by its listening port** (from `ss -ltnp`), never by process name, so the live service can
never be matched by mistake. With `--test` it stops after that and does not install. The test
instance shares the database with the live one but cannot take the writer lock (SPEC-0080
REQ-006), so it never double-ingests.

### SPEC-0082-REQ-010: Standard library elsewhere

Everything outside the store (save decoding, metrics, S3 backup with its own SigV4, the
secret-sync script) shall remain standard library only. The secret-sync script runs on
`/usr/bin/python3`, not the venv, so it works before the venv exists.

## Degraded behaviour

| Condition | Behaviour |
|---|---|
| `uv` missing | install stops before touching units; the running service is unchanged |
| package index unreachable on a fresh install | `uv sync` fails, install stops; a reinstall with a warm `UV_CACHE_DIR` works offline |
| `uv.lock` out of date with `pyproject.toml` | `--frozen` installs what the lock says; `uv lock --check` in development catches the drift |
| system Python upgraded in place (minor version) | the venv's interpreter link breaks; rerun install (it rebuilds) |
| `db.env` absent | service runs; history APIs 503 (SPEC-0081) |
| `s3.env` absent | save backup disabled; nothing else affected |

## Verification

1. `sudo NMS_USER=<user> ./install.sh` ends with "nms-exporter@<user> running".
2. `/opt/nms-exporter/.venv/bin/python -c 'import psycopg; print(psycopg.__version__)'` prints
   the locked version, and `readlink -f /opt/nms-exporter/.venv/bin/python` resolves to the
   system interpreter, not a uv-managed one.
3. `systemctl show nms-exporter@<user> -p ExecStart` names the venv interpreter.
4. `sudo stat -c '%a %U' /etc/nms-exporter /etc/nms-exporter/db.env` prints `700 root` and
   `600 root`.
5. `curl -s localhost:9877/metrics | grep nms_db_up` is `1` with `db.env` present and the
   database up.
6. `scripts/dev-deploy.sh --test` prints `errors: none` and a metric count, and leaves nothing
   listening on 9878 afterwards (probe the test instance by hand while it runs, not after).

## Lessons

See `spec/lessons/SPEC-0099-storage-deploy-lessons.md`: a held package-manager lock (an
unattended dist-upgrade) blocked installing uv; download uv's standalone binary to test rather
than waiting. And `--test` stops its own exporter at the end, so probing 9878 afterwards finds
nothing.
