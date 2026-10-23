---
id: SPEC-0099
title: 'Storage and deploy lessons'
description: 'Gotchas from moving the history to Postgres, packaging the exporter, giving a host machine identity for secrets and exposing the map, ordered by time saved.'
type: lesson
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
related_specs: [SPEC-0080, SPEC-0081, SPEC-0082, SPEC-0083, SPEC-0084, SPEC-0085]
---

# SPEC-0099: Storage and deploy lessons

## Lesson: the reverse proxy hides who is logging in

**What happened.** Binding the gaming PC's machine role to its LAN IP did nothing useful: every
request reached the secret store from the reverse proxy's pod, so the store saw a pod address.
After the store was told to trust `X-Forwarded-For` from the pod network, logins from the gaming
PC still did not show its LAN IP. They showed the node's **pod-network gateway address** (its
overlay interface), because the gaming PC is also a cluster node and its own kube-proxy routes
traffic to the proxy's load-balancer IP, leaving with the overlay address. A second node behaved
the same way with its own gateway address.

**The rule.** Never guess the source address; read it from the audit log
(`request.remote_address` on `auth/approle/login`). Trust `X-Forwarded-For` only from the proxy's
network, make sure the proxy overwrites the header (test with a forged one), bind the gateway
address plus the bridge `.1` and LAN IP as fallbacks, and confirm stability with 20 logins all
from one address. Do not bind a non-node host this way: it arrives as whichever node announces
the LB IP, which changes on failover.

## Lesson: an unset PGHOST becomes a Unix-socket error

**What happened.** The first run of the exporter had no `db.env`. `/api/map` called into psycopg
anyway; libpq with no `PGHOST` tried the default local Unix socket and reported a socket error,
which looked like a broken database rather than a missing configuration. (The unit's
`RestrictAddressFamilies` excludes `AF_UNIX`, which makes the message even less obvious.)

**The rule.** Check "configured" (`PGHOST` non-empty) before any driver call, in every path that
touches the store, and say "not configured: no PGHOST, the secret sync writes db.env" in the
503 body. SPEC-0081 REQ-002 and REQ-004.

## Lesson: commit the offset in the same transaction as the rows

**What happened.** The ingest design had to survive the exporter being restarted at any moment
(every deploy restarts it) and the database dropping mid-pass.

**The rule.** One transaction per file chunk: insert the rows and upsert `ingest_file.pos`
together. Only read up to the last newline. Then a crash or lost connection at any point either
committed both or neither, and the next pass rereads exactly the uncommitted bytes. Do not keep
the offset in a local file; it can disagree with the database.

## Lesson: most visor markers are repeats

**What happened.** One day's visor file was 17.8 MB. 40,468 of its 41,760 `marker_add` lines were
the game re-adding markers it already had, every time an area loaded.

**The rule.** De-duplicate on ingest by `(type, name, rel rounded to 5 m cells)`, rebuilding the
seen-set from the table after any error. Keep everything else (label, mark, visor events)
verbatim.

## Lesson: the external secret must exist before the cluster is pushed

**What happened.** The database cluster's bootstrap needs its credentials Secret, which the
external-secrets operator builds from the secret store. Pushed before the store entry existed, the
bootstrap pod sat in `CreateContainerConfigError` with nothing pointing at the cause. Once the
entry was created, the operator synced the Secret within its refresh interval and the pod started
on its own.

**The rule.** Create the secret-store entry (and the host policy that reads it) first, then push
the database manifests. If you see `CreateContainerConfigError` on a bootstrap pod, look at the
ExternalSecret's status before anything else. Also: the Secret must be type
`kubernetes.io/basic-auth` for CloudNativePG to read it.

## Lesson: `bao unwrap` sends your ambient token

**What happened.** Unwrapping a response-wrapped secret ID with `bao unwrap <wrapping token>`
failed 403 as if the wrapping token were bad. The CLI had sent the ambient token
(`BAO_TOKEN` / `~/.vault-token`), which was stale.

**The rule.** Unset the ambient token before unwrapping, or unwrap with
`curl -X POST -H "X-Vault-Token: <wrapping token>" $ADDR/v1/sys/wrapping/unwrap`. Do it with
`umask 077` straight into the credentials file.

## Lesson: Argo CD silently drops EndpointSlices

**What happened.** The map's edge route (selectorless Service + EndpointSlice + Ingress) synced
green, but the slice never existed: Argo CD's default `resource.exclusions` include
`discovery.k8s.io/EndpointSlice`. The Ingress answered 503 "no ready endpoints" while the
Service looked perfect.

**The rule.** Apply EndpointSlices for external backends imperatively and re-apply after any
change. Check with `kubectl get endpointslice`, not with the GitOps UI.

## Lesson: a GitOps app can be "Synced" on an old commit

**What happened.** After pushing the edge route, the app reported Synced, but at the previous
revision; the new Ingress did not exist. A hard refresh and a manual sync fixed it.

**The rule.** After a push, compare the app's synced revision with the commit you pushed before
debugging anything downstream.

## Lesson: OutOfSync with an empty diff on the database Cluster

**What happened.** The GitOps tool showed the CloudNativePG `Cluster` OutOfSync with an empty
diff. The cause was an operator-defaulted key inside an atomic list (the additional
load-balancer service's `updateStrategy`), which server-side apply treats as a change to the
whole list.

**The rule.** State operator defaults explicitly in atomic lists (`updateStrategy: patch`). If an
empty-diff OutOfSync remains, it is cosmetic; do not "fix" it by letting the tool prune or
replace the Cluster.

## Lesson: audit devices fail closed

**What happened.** Adding an audit device was needed so machine reads are visible. The store
refuses every request if no audit device can write.

**The rule.** Declare the audit device in the configuration file (so it survives rebuilds) and
point it at stdout. Know that a full log partition on the store's node now stops every secret
read, which shows up as every sync failing at once.

## Lesson: a held package-manager lock blocked installing uv

**What happened.** Installing `uv` from the OS package manager waited on a lock held by an
unattended dist-upgrade that was going to take a long time.

**The rule.** To test, download uv's standalone binary into a scratch directory and use that; the
install script only needs `uv` on `PATH`. Install the OS package later.

## Lesson: `dev-deploy --test` stops its own exporter

**What happened.** After `scripts/dev-deploy.sh --test` finished, probing port 9878 found
nothing and looked like a failure. The script deliberately stops the test exporter at the end
(it finds it by port, so it can never kill the live service).

**The rule.** Probe the test instance by hand while it runs (start it yourself with
`NMS_EXPORTER_PORT=9878 NMS_STATE=<scratch> .venv/bin/python nms_exporter.py`), or read the
script's own output. The test instance reads the shared database but cannot take the writer
lock, so it never double-ingests.

## Lesson: rotating the database password is two steps, in order

**What happened.** The operator reads the bootstrap Secret once, at creation. Changing the value
in the secret store afterwards changes nothing in the database, but the gaming PC's sync picks up
the new value and the exporter then fails to log in.

**The rule.** `ALTER ROLE nms PASSWORD ...` first, then update the secret store.

## Lesson: if WAL archiving breaks, the volume fills

**What happened.** With continuous archiving on, a broken archive target makes `pg_wal` grow
until the volume is full, which stops the database.

**The rule.** Watch `select archived_count, failed_count, last_failed_wal from pg_stat_archiver;`
(and alert on a rising `failed_count`).

## Lesson: a node that rejoins can change its pod CIDR

**What happened.** The machine-role binding uses the node's pod-network gateway address. A node
deleted and rejoined can get a different pod CIDR, and its logins then fail with
`source address "<addr>" unauthorized by CIDR restrictions on the role`.

**The rule.** If that error appears, read the node's `spec.podCIDR` and re-bind the role
(`secret_id_bound_cidrs` and `token_bound_cidrs`). The trailing `%!w(<nil>)` in OpenBao's error
is a harmless formatting bug.
