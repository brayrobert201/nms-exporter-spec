---
id: SPEC-0083
title: 'The managed Postgres cluster'
description: 'A two-instance Postgres from a Kubernetes operator, kept off the gaming PC, exposed on a LAN load-balancer address with TLS, backed up by daily snapshots plus a WAL archive, with credentials from the secret store.'
type: snapshot
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0080]
supersedes: []
superseded_by: []
related_specs: [SPEC-0081, SPEC-0084, SPEC-0090, SPEC-0099]
screenshots: []
---

# SPEC-0083: The managed Postgres cluster

## Summary

The history store (SPEC-0080) is a dedicated Postgres database run by a Postgres operator in the
home Kubernetes cluster (the original uses CloudNativePG; any operator with instances, a
load-balancer service, scheduled backups and WAL archiving will do). It is not run on the gaming
PC: that machine reboots whenever its owner likes and is the database's only client, so an
instance there would be down exactly when it is needed. Two instances give failover across a node
drain or operator restart, which matters because the map has no file fallback (SPEC-0081). The
exporter reaches the read-write primary from outside the cluster through a LAN-only
load-balancer IP. Credentials live once in the secret store; the cluster gets them through an
external-secrets operator and the gaming PC through its machine role (SPEC-0084), so both sides
always agree.

A rebuilder without a cluster can run a single Postgres 17 anywhere on the LAN that is not the
gaming PC; the exporter only needs the libpq variables. The requirements below that are about
the operator are then replaced by "back it up and keep it running".

## Inputs and outputs

| Item | Value |
|---|---|
| Postgres | 17.x (operator's standard image) |
| instances | 2 (primary + streaming replica) |
| database / owner | `nms` / `nms` (created by the operator at bootstrap) |
| storage | 5 GiB per instance on a single-replica volume class (Postgres replication is the redundancy); volume is MBs per day |
| client endpoint | `<LAN LB IP>:5432`, a load-balancer Service selecting the **rw** (primary) pods |
| TLS | required by the client (`PGSSLMODE=require`); server certificate from the operator's self-signed CA, not verified |
| backups | daily volume snapshot at 05:15 to an off-cluster backup target; continuous WAL archive to object storage, 30-day retention, gzip |
| credentials | one secret-store entry with keys `db_username`, `db_password` |

## Requirements

### SPEC-0083-REQ-001: Not on the client host

The database instances shall be scheduled only on nodes other than the gaming PC (a required
node anti-affinity on its hostname, if the gaming PC is itself a cluster node).

### SPEC-0083-REQ-002: Two instances

The cluster shall run 2 instances so that a single pod, node or operator restart fails over
rather than taking the map down.

### SPEC-0083-REQ-003: Bootstrap from the shared secret

The operator shall create database `nms` owned by role `nms`, with the role's password taken
at bootstrap from a Kubernetes Secret of type `kubernetes.io/basic-auth` (`username`,
`password`). That Secret shall be produced by an external-secrets operator from the secret-store
entry (`db_username`, `db_password`), refreshed hourly, through a cluster-wide secret store that
authenticates with Kubernetes auth. Example (External Secrets Operator):

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: nms-db-creds, namespace: <ns>}
spec:
  secretStoreRef: {kind: ClusterSecretStore, name: <store>}
  refreshInterval: 1h
  target:
    name: nms-db-creds
    template:
      engineVersion: v2
      type: kubernetes.io/basic-auth   # the operator reads username/password from this type
      data: {username: "{{ .db_username }}", password: "{{ .db_password }}"}
  data:
    - {secretKey: db_username, remoteRef: {key: <db secret>, property: db_username}}
    - {secretKey: db_password, remoteRef: {key: <db secret>, property: db_password}}
```

### SPEC-0083-REQ-004: Rotation order

The bootstrap secret is read once, at cluster creation. To rotate the password, the operator of
the system shall first `ALTER ROLE nms PASSWORD ...` in the database, then update the secret-store
entry; the gaming PC picks it up on its next sync (SPEC-0084) and restarts the exporter. Changing
only the secret store does not rotate anything and breaks the client.

### SPEC-0083-REQ-005: LAN load-balancer for the primary

The cluster shall expose an additional Service of type `LoadBalancer` selecting the read-write
primary, on a fixed address from the LAN load-balancer pool (e.g. MetalLB with a
`loadBalancerIPs` annotation). That address shall be reachable from the LAN only, never
forwarded from the internet. With CloudNativePG this is `spec.managed.services.additional` with
`selectorType: rw`; state `updateStrategy: patch` explicitly (it is the server default) so the GitOps
diff stays clean.

### SPEC-0083-REQ-006: TLS required, CA not verified

The client shall connect with `PGSSLMODE=require`. The server certificate is issued by the
operator's own self-signed CA and names the in-cluster service, not the LB IP, so `verify-full`
would fail. On a LAN with a fixed LB address this is accepted: traffic is encrypted and the
password is not sent in clear; an attacker able to impersonate the LB IP on the LAN is out of
scope. A rebuilder wanting verification exports the operator's CA to the client and uses
`verify-ca`.

### SPEC-0083-REQ-007: Daily snapshot

A scheduled backup shall take a volume snapshot of the database daily (05:15 in the original,
chosen to avoid other backups' slots; the operator's cron has six fields:
`"0 15 5 * * *"`) to a snapshot class whose target is off the cluster.

### SPEC-0083-REQ-008: WAL archive

The cluster shall archive WAL continuously to object storage (CloudNativePG: the barman-cloud
plugin with `isWALArchiver: true`, an `ObjectStore` with `retentionPolicy: 30d`, gzip for WAL
and data, `maxParallel: 2`). Object-storage credentials come from the secret store like
everything else.

### SPEC-0083-REQ-009: Protect against GitOps deletion

The database's namespace and Cluster objects shall carry the GitOps tool's "never prune, never
delete" options (Argo CD: `argocd.argoproj.io/sync-options: Prune=false,Delete=false`), and the
application shall sync with `prune: false`, so that removing a manifest by mistake never takes the
volumes with it.

### SPEC-0083-REQ-010: Secret entry exists before the cluster

The secret-store entry shall be created before the cluster manifests are pushed. (If not, the
operator's bootstrap pod waits in `CreateContainerConfigError` for the Secret; it recovers on its
own once the entry exists, but nothing says why meanwhile.)

## Degraded behaviour

| Condition | Effect |
|---|---|
| primary pod lost | operator promotes the replica; the LB Service follows the rw selector; the exporter's next 2 s pass reconnects (SPEC-0080 REQ-014) |
| both instances down | SPEC-0081: 503s, banner, `nms_db_up 0`; day files keep growing and are ingested on recovery |
| WAL archiving broken | `pg_wal` grows until the volume fills, then the database stops; watch `failed_count` |
| backup target down | snapshots fail; data unaffected; WAL archive still gives point-in-time recovery |
| secret store down | running database and client unaffected; only rotation and fresh bootstrap wait |
| LB address taken by something else | client connects to the wrong thing or times out; pick the address from the pool's free range and record it |

## Verification

1. From the gaming PC, with `db.env` sourced:
   `psql -tAc 'select ssl from pg_stat_ssl where pid = pg_backend_pid()'` prints `t` (the
   session is encrypted). The client's `PGSSLMODE=require` is what enforces TLS; whether the
   server also refuses plain connections depends on the operator's `pg_hba` defaults, so check
   with `sslmode=disable` and add a `hostssl`-only rule if you want the server to insist too.
2. `kubectl get pods -o wide` (database namespace): two instance pods, neither on the gaming PC.
3. WAL archiving:
   ```bash
   kubectl -n <ns> exec <cluster>-1 -c postgres -- \
     psql -U postgres -tAc "select archived_count, failed_count, last_failed_wal from pg_stat_archiver;"
   ```
   `archived_count` rises over time and `failed_count` is 0 (or not rising).
4. The day after creation, a completed backup object exists for the scheduled backup.
5. Fail over (delete the primary pod): within a minute the exporter's `nms_db_up` returns to 1
   and no trail lines are missing (`SELECT count(*)` against the day file's line count).
6. In the GitOps tool the application is Healthy. (It may show the Cluster OutOfSync with an
   empty diff; see lessons. That is cosmetic.)

## Lessons

See `spec/lessons/SPEC-0099-storage-deploy-lessons.md`: create the secret before pushing the
cluster, and an OutOfSync Cluster with an empty diff is an operator-defaulted key inside an
atomic list, not a real drift.
