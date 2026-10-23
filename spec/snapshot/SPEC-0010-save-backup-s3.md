---
id: SPEC-0010
title: 'Save backup to an S3-compatible bucket'
description: 'Every save and its manifest uploaded under a key carrying the file''s own mtime, the newest save also at fixed current/ keys, tiered retention, with a hand-rolled SigV4 client and nothing that can raise into the exporter.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0001, SPEC-0002]
supersedes: []
superseded_by: []
related_specs: [SPEC-0011, SPEC-0012, SPEC-0014, SPEC-0019, SPEC-0084]
screenshots: []
---

# SPEC-0010: Save backup to an S3-compatible bucket

## Summary

The game keeps a handful of slots and overwrites them; Steam Cloud syncs whatever is newest,
including a mistake. The exporter already notices every new save, so it also uploads each one,
with its manifest, to an S3-compatible bucket under a key that carries the file's own mtime.
The store used has no object versioning, so each version is its own key (SPEC-0014). The newest
save of any slot is also copied to three fixed keys under `current/` so a restore or another tool
never searches. A retention pass thins old versions. The client is ~100 lines of standard
library: SigV4 signing, PUT, DELETE and ListObjectsV2.

## Inputs and outputs

Configuration (environment; the unit reads an optional root-owned, mode-600 env file):

| Variable | Required | Default |
|---|---|---|
| `NMS_S3_ACCESS_KEY` | yes | - |
| `NMS_S3_SECRET_KEY` | yes | - |
| `NMS_S3_ENDPOINT` | no | an S3 endpoint on loopback (`http://<loopback>:3900` in the reference) |
| `NMS_S3_BUCKET` | no | a fixed bucket name |
| `NMS_S3_REGION` | no | the store's region name (used only in signing) |

The keys come from the secret store; how they reach the env file is the `storage-deploy` area.

Files read: every non-empty `save*.hg` and `accountdata.hg` in the save directory, and each one's
`mf_<name>` manifest when present.

Object layout:

```
<st_dir>/<stem>/<YYYYmmddTHHMMSSZ>/<stem>.hg        e.g. st_<id>/save3/20261022T091203Z/save3.hg
<st_dir>/<stem>/<YYYYmmddTHHMMSSZ>/mf_<stem>.hg
current/save.hg
current/mf_save.hg
current/info.json
```

`<st_dir>` is the save directory's basename, `<stem>` the file name without `.hg`, the stamp the
file's mtime in UTC. `info.json`:

```json
{"file": "save3.hg", "manifest": "mf_save3.hg", "slot_file_stem": "save3",
 "mtime": 1792713084.9, "mtime_utc": "20261022T091203Z", "bytes": 10012345,
 "sha256": "<hex>", "version_key": "st_<id>/save3/20261022T091203Z/save3.hg",
 "restore": "copy current/save.hg to save3.hg and current/mf_save.hg to mf_save3.hg"}
```

Status: `/json backup = {enabled, bucket, endpoint, reason, uploads, errors, last_upload,
last_key, last_prune, last_error}` and the `nms_backup_*` metrics (SPEC-0002).

## Requirements

### SPEC-0010-REQ-001: Disabled without keys

If either key is unset, the backup shall be disabled, do nothing, and report
`reason: "NMS_S3_ACCESS_KEY / NMS_S3_SECRET_KEY not set"` and `nms_backup_enabled 0`.

### SPEC-0010-REQ-002: Cheap kick on every refresh

On every exporter `refresh()` the backup's `kick(save_dir)` shall: return at once if disabled;
compute whether a prune is due (every **6 h**) and whether any file is pending; if neither,
return; otherwise start one background thread unless one is already running. `kick` never blocks
on the network.

### SPEC-0010-REQ-003: Pending and settle

A file is pending when its mtime differs from the mtime this process last uploaded for it, it is
non-empty, and it is at least **2 s** old (the game may still be writing a younger file).

### SPEC-0010-REQ-004: Versioned upload

For each pending file: PUT the file at `<st_dir>/<stem>/<stamp>/<name>`, then its manifest at
`.../mf_<name>` if it exists; record the mtime as uploaded; count the upload; set `last_upload`
and `last_key`. Keys are derived from mtime only, so a re-upload after a restart rewrites the same
objects (idempotent) instead of duplicating them.

### SPEC-0010-REQ-005: current/

After uploads, among settled non-empty `save*.hg`, take the newest by mtime; if it differs from
the `(path, mtime)` last written to `current/`, PUT it to `current/save.hg`, its manifest to
`current/mf_save.hg`, and `current/info.json` as above. Overwritten in place.

### SPEC-0010-REQ-006: Retention

When a prune is due, list `<st_dir>/` (ListObjectsV2 with continuation tokens), group keys of
exactly four path parts by stem then stamp, and keep per stem:

- every stamp up to **7 days** old;
- the newest stamp per hour (`stamp[:11]`) from 7 to **30 days**;
- the newest stamp per day (`stamp[:8]`) after that.

Delete every key of every other stamp. `current/` is never pruned.

### SPEC-0010-REQ-007: SigV4

Requests shall be path-style (`/<bucket>/<key>`, each key segment percent-encoded with safe set
`-_.~`) and signed with AWS Signature Version 4, service `s3`:

```
canonical_query   = "&".join(quote(k)=quote(v) for sorted query items)
headers           = host, x-amz-date, x-amz-content-sha256 (lowercased, whitespace collapsed)
canonical_request = METHOD \n path \n canonical_query \n "k:v\n"... \n signed_header_names \n sha256(body)
string_to_sign    = "AWS4-HMAC-SHA256" \n amz_date \n date/region/s3/aws4_request \n sha256(canonical_request)
k_signing         = HMAC(HMAC(HMAC(HMAC("AWS4"+secret, date), region), "s3"), "aws4_request")
Authorization     = AWS4-HMAC-SHA256 Credential=<access>/<scope>, SignedHeaders=<names>, Signature=hex(HMAC(k_signing, sts))
```

`content-length` is added after signing and not signed. PUT expects 200, DELETE 200/204, LIST 200
(XML parsed with the namespace taken from the root tag). Any other status raises with the first
200 bytes of the body.

### SPEC-0010-REQ-008: Never raise into the exporter

The whole run shall be inside one `try`: an exception increments `errors`, sets `last_error`
(`"<Type>: <message>"`) and ends the run; a later successful run clears `last_error`. The
running flag is cleared in `finally`. Uploads already done in a failed run stay recorded.

### SPEC-0010-REQ-009: Restore procedure

To restore: close the game, turn Steam Cloud off for the game, copy both files of a version (or
`current/save.hg` and `current/mf_save.hg` under the names in `info.json`) back into the save
directory.

## Degraded behaviour

| Condition | Result |
|---|---|
| No keys | Disabled, reason in `/json`, `nms_backup_enabled 0`. Exporter unaffected. |
| Store down / wrong keys | Each run fails: `nms_backup_errors_total` rises, `last_error` explains; pending files stay pending and upload when it recovers (mtime unchanged, so the same keys). |
| Exporter restarted | `_seen` is empty, so every current slot re-uploads once to the same keys (harmless); `current/` rewritten once. |
| Game writing during a run | Files younger than 2 s wait for the next kick. |
| Prune fails mid-way | Some old versions linger until the next prune 6 h later. |

## Verification

1. Check the signer against AWS's published SigV4 examples (the reference was).
2. With keys set, save in game; within a scrape `/json backup.uploads` increments and
   `last_key` names the new stamp. List the bucket: the version key and `mf_` sibling exist;
   `current/info.json` sha256 equals `sha256sum` of the local file.
3. Restart the exporter: no new stamps appear, only rewrites.
4. Unit-test `to_keep` with synthetic stamps at 1 h, 3 d, 10 d (two in one hour), 40 d (two in
   one day): exactly one of each pair survives.
5. Point `NMS_S3_ENDPOINT` at a closed port: `/json` and `/metrics` still answer instantly;
   `nms_backup_errors_total` rises.

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): no object versioning in the store
means versions must be keys; deriving keys from mtime keeps uploads idempotent across restarts.
