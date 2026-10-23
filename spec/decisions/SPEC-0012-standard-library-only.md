---
id: SPEC-0012
title: 'Standard library only on the save side'
description: 'The save decoder, exporter, backup client and wall-screen renderer use only the Python standard library: no pip, no build step, nothing to break on an unattended host.'
type: decision
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0001, SPEC-0002, SPEC-0009, SPEC-0010, SPEC-0011, SPEC-0091]
---

# SPEC-0012: Standard library only on the save side

## Context

The exporter runs unattended for weeks on a gaming PC whose system Python is upgraded by the
distribution, not by us. Every dependency is something that can fail to install, break on a
Python upgrade, or need a virtualenv the service must find. The obvious dependencies were all
available as packages:

- `lz4` for the save's compressed chunks;
- `prometheus_client` for the exposition format;
- `boto3` for the S3 backup;
- an HTTP framework for the server, `requests` for the conky renderer;
- `python-xlib` for the click watcher.

## Decision

Everything that reads the save, serves `/metrics` and `/json`, backs saves up, and renders the
wall screens uses only the standard library:

| Need | Done with |
|---|---|
| LZ4 block decode | ~40-line pure-Python decoder (SPEC-0001) |
| HTTP server | `http.server.ThreadingHTTPServer` |
| Prometheus text format | string formatting with an escaping helper (SPEC-0002-REQ-009) |
| S3 | `http.client` + `hmac`/`hashlib` SigV4 + `xml.etree` for listings (SPEC-0010) |
| Fetching `/json` | `urllib.request` |
| X11 clicks | `ctypes` into libX11 and libXi |
| Grafana dashboards | generator scripts writing JSON |

Tools that run by hand (the game-file extractor) may use external binaries; they are not on the
service's path.

## Alternatives considered

- **Pin dependencies in a virtualenv.** Works, but adds a build step to install, a lock file to
  maintain, and a failure mode on every Python minor upgrade. Later, the map's Postgres history
  did need a driver (`psycopg`), and the project accepted a locked environment for that; the save
  side still imports nothing from it, so the screens and metrics survive that environment
  breaking.
- **`lz4` package.** Faster, but decode is a few seconds once per save, which nobody waits on.
- **`boto3`.** Hundreds of files to sign four request types.

## Consequences

- Install is a copy plus a unit file; nothing to download at install time.
- More code is ours: the LZ4 decoder and SigV4 signer must be tested against reference vectors
  (the signer was checked against AWS's published examples).
- Decode is slower than native LZ4 (seconds, exported as a metric).
- Revisit if decode time grows past the scrape interval, or if the backup needs multipart uploads
  (saves are ~10 MB today, well under single-PUT limits).
