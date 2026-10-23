---
id: SPEC-0014
title: 'Save backup as mtime-keyed versions plus a fixed current/ key'
description: 'Each save version is its own object key derived from the file''s mtime, and the newest save is also overwritten at current/save.hg, because the object store has no versioning.'
type: decision
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0010]
---

# SPEC-0014: Save backup as mtime-keyed versions plus a fixed current/ key

## Context

The game has a few save slots and overwrites them; Steam Cloud faithfully syncs a bad save. A
backup must keep history (to roll back hours or days) and make "the latest save" trivial to
fetch for a restore or for another tool. The available bucket is on a self-hosted S3-compatible
store that does **not** implement S3 object versioning. The exporter restarts (deploys, reboots)
and must not create duplicates or lose track of what it uploaded.

## Decision

- **Versions are keys.** Each file is stored at `<st_dir>/<stem>/<UTC mtime stamp>/<name>`, with
  its `mf_` manifest beside it. The key is a pure function of the file's mtime, so uploading the
  same version twice writes the same objects: idempotent, no state needed across restarts.
- **One fixed latest.** The newest save of any slot is also written to `current/save.hg`,
  `current/mf_save.hg` and `current/info.json` (which slot, mtime, sha256, the versioned key,
  restore instructions), overwritten in place.
- **Tiered retention** in the same process: all for 7 days, newest per hour to 30 days, newest
  per day after (pruned every 6 h).
- **Settle 2 s** before reading a file, as the decoder does.

## Alternatives considered

- **Bucket versioning** on one key per slot: not available on the store; would also tie restore
  to version-id APIs.
- **Content-hash keys**: idempotent too, but unreadable when browsing and needs the hash before
  the key; mtime stamps sort chronologically and say when.
- **A separate backup tool on a timer (rclone/restic)**: another moving part, another schedule
  that can miss the 2 s settle window, and it would not know which slot is newest.
- **Only current/**: no history, which is the point.

## Consequences

- Restore is "copy two files back" from either a version folder or `current/`.
- A restart re-uploads each slot once (same keys) - a few tens of MB, harmless.
- Storage grows with play time until retention thins it; roughly 10 MB per version.
- Saves are uploaded unencrypted; the bucket's access policy is the protection. Revisit if the
  bucket moves off-site.
