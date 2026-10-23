---
id: SPEC-0084
title: 'Machine identity for secrets'
description: 'How the gaming PC reads its database login and backup key from the secret store unattended: an AppRole-style role with a non-expiring secret bound to the host''s source address, a timer that logs in, writes root-only env files, restarts on change and revokes its token.'
type: snapshot
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0082]
supersedes: []
superseded_by: []
related_specs: [SPEC-0083, SPEC-0092, SPEC-0099, SPEC-0010]
screenshots: []
---

# SPEC-0084: Machine identity for secrets

## Summary

The exporter needs two secrets: the Postgres login (SPEC-0083) and, for the save backup, an
S3 access key (SPEC 0001–0019). Neither is committed or typed in by hand. They live in a secret
store with AppRole-style machine roles (OpenBao or HashiCorp Vault). Inside the Kubernetes
cluster, workloads authenticate with the store's Kubernetes auth method and need nothing from
this spec. The gaming PC is a bare-metal host, so it gets its own role: a `role_id` and a
**non-expiring** `secret_id` stored root-only on the host, and the role only accepts logins
whose source address is that host's. A systemd timer logs in, reads exactly the secrets its
policy grants, rewrites the env files only when their content changed, restarts the exporter
only then, and revokes its token before exiting.

The source-address binding is the subtle part. The store sits behind a reverse proxy, so it sees
the proxy's address unless told to trust `X-Forwarded-For`, and a host that is also a cluster
node does not arrive with its LAN IP at all.

## Inputs and outputs

| Item | Value |
|---|---|
| store address | `NMS_BAO_ADDR` (HTTPS URL of the store behind the LAN reverse proxy) |
| credentials | `/etc/nms-exporter/bao/role_id`, `/etc/nms-exporter/bao/secret_id` (root, 600) |
| secrets read | the S3 backup entry (`access_key_id`, `secret_access_key`) and the database entry (`db_username`, `db_password`); both KV v2 |
| outputs | `/etc/nms-exporter/s3.env` (`NMS_S3_ACCESS_KEY`, `NMS_S3_SECRET_KEY`), `/etc/nms-exporter/db.env` (the libpq variables, SPEC-0082) |
| script overrides | `NMS_BAO_ADDR`, `NMS_BAO_DIR`, `NMS_S3_ENV`, `NMS_DB_ENV`, `NMS_DB_HOST` (the database LB address), `NMS_EXPORTER_UNIT` |
| unit | `nms-bao-sync.service` (oneshot, root) + `nms-bao-sync.timer` |

### Role settings (normative)

```bash
bao write auth/approle/role/<host> \
  token_policies=<host policy> \
  secret_id_ttl=0 secret_id_num_uses=0 \
  token_ttl=900 token_max_ttl=3600 \
  secret_id_bound_cidrs=<addr1>/32,<addr2>/32,<lan ip>/32 \
  token_bound_cidrs=<addr1>/32,<addr2>/32,<lan ip>/32
```

- `secret_id_ttl=0`, `secret_id_num_uses=0`: the secret never expires and is reusable. A
  machine that is off for a month must still be able to log in.
- `token_ttl=900` (15 min), `token_max_ttl=3600`: each run's token is short-lived, and it is
  revoked explicitly anyway.
- Both CIDR lists hold the addresses the host is **seen as** by the store (below), each `/32`.
  A copied `secret_id` is useless from any other address.

### Policy (normative shape)

```hcl
# read the save-backup key and the database login; nothing else
path "<kv mount>/data/<s3 backup entry>" { capabilities = ["read"] }
path "<kv mount>/data/<database entry>"  { capabilities = ["read"] }
```

## Requirements

### SPEC-0084-REQ-001: Login, read, revoke

The sync script shall `POST /v1/auth/approle/login` with `{"role_id", "secret_id"}` read from the
credentials directory, take `auth.client_token`, `GET` each secret with header
`X-Vault-Token`, and in a `finally` block `POST /v1/auth/token/revoke-self` with an empty JSON
body, ignoring errors from the revoke. The token shall never be written to disk.

The same shape in shell, for other machine jobs on the same pattern (revoke on any exit):

```bash
TOKEN=$(curl -sS --fail -X POST "$ADDR/v1/auth/approle/login" \
  -d "{\"role_id\":\"$(cat "$DIR/role_id")\",\"secret_id\":\"$(cat "$DIR/secret_id")\"}" \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["auth"]["client_token"])')
hdr=(-H "X-Vault-Token: $TOKEN")
trap 'curl -sS -X POST "${hdr[@]}" "$ADDR/v1/auth/token/revoke-self" -d "{}" >/dev/null 2>&1 || true' EXIT
data=$(curl -sS --fail "${hdr[@]}" "$ADDR/v1/<kv mount>/data/<entry>")
```

### SPEC-0084-REQ-002: A missing grant is not fatal

If reading the database entry fails with an HTTP error (not created yet, or not yet in the
policy), the script shall print `<entry>: HTTP <code> (not created yet, or not in the policy);
db.env left as is` and carry on with the S3 entry.

### SPEC-0084-REQ-003: Write only on change, atomically, root-only

For each env file the script shall compare the full new content with the existing file and do
nothing if equal. Otherwise it shall write `<path>.tmp` created with mode 0600 and
`os.replace` it over the file. Each file starts with a comment saying it is generated and must
not be edited by hand.

### SPEC-0084-REQ-004: Restart only when something changed

If at least one file changed, the script shall run `systemctl try-restart <exporter unit>`
(restart only if running) and print which files changed. If none changed it shall print
"unchanged" and exit 0.

### SPEC-0084-REQ-005: Database env content

`db.env` shall contain exactly `PGHOST=<NMS_DB_HOST>`, `PGPORT=5432`, `PGDATABASE=nms`,
`PGUSER=<db_username>`, `PGPASSWORD=<db_password>`, `PGSSLMODE=require`, one per line.

### SPEC-0084-REQ-006: Standard library, system Python

The script shall use only the Python standard library (`urllib.request`, `json`, `subprocess`)
and run under `/usr/bin/python3`, with a 20 s timeout per request. On any unhandled error it
shall print `sync failed: <type>: <message>` to stderr and exit 1.

### SPEC-0084-REQ-007: Timer

The timer shall fire 2 minutes after boot and every 6 hours after the last run, with up to 10
minutes random delay, `Persistent=true`. The service shall be `Type=oneshot`, `PrivateTmp=yes`,
`ProtectHome=yes`, `NoNewPrivileges=yes`, after `network-online.target`.

### SPEC-0084-REQ-008: Source addresses as the store sees them

The store's listener shall trust `X-Forwarded-For` **only** from the reverse proxy's network
(OpenBao/Vault: `x_forwarded_for_authorized_addrs = "<proxy/pod network CIDR>"`,
`x_forwarded_for_reject_not_present = "false"` so in-cluster clients that call the store's
Service directly, with no header, still work). The reverse proxy shall overwrite
`X-Forwarded-For` with the address it saw and trust no incoming `X-Forwarded-*` (Traefik's
default without `forwardedHeaders.trustedIPs`).

### SPEC-0084-REQ-009: Which address a cluster node arrives as

Where the host is also a Kubernetes node, the role shall be bound to the host's **pod-network
gateway address** (its overlay interface address, e.g. flannel's `.0` of the node's pod CIDR),
plus the pod bridge gateway (`.1`) and the LAN IP as fallbacks. Reason: the node's request to
the proxy's load-balancer IP is routed by the node's own kube-proxy and leaves with the node's
pod-network address, so that is what the proxy writes into `X-Forwarded-For`. It is stable per
node, but it is not the LAN IP. Determine it empirically from the audit log
(`request.remote_address` on the host's login) rather than by reasoning.

### SPEC-0084-REQ-010: Non-node hosts

Where the host is not a cluster node, its requests arrive as whichever node the load balancer is
announcing the proxy's IP from, which moves on failover. Such a host shall not be address-bound
this way without re-testing (it would need `externalTrafficPolicy: Local` on a dedicated LB IP
so the real source survives).

### SPEC-0084-REQ-011: Audit log

The store shall have an audit device declared in its configuration file (not via the API, so it
survives a rebuild) writing every request and response to stdout, with secrets HMAC'd
(`log_raw = "false"`), so the pod log and the log aggregator record which machine read what.
OpenBao/Vault syntax:

```hcl
audit "file" "stdout" {
  description = "all requests and responses, to the pod log"
  options { file_path = "stdout"  log_raw = "false" }
}
```

Audit devices fail closed: if no audit device can write, the store refuses every request. stdout
is the safest target, but a full node log partition would take the store down with it.

### SPEC-0084-REQ-012: Minting the secret ID

An administrator shall mint the `secret_id` once, response-wrapped
(`bao write -wrap-ttl=1h -f auth/approle/role/<host>/secret-id`), hand the wrapping token to
the host, and unwrap it there with `umask 077` straight into the credentials file. Unwrap with no
ambient token set (or with `curl -X POST -H "X-Vault-Token: <wrapping token>"
$ADDR/v1/sys/wrapping/unwrap`). Remove an old one with
`bao write auth/approle/role/<host>/secret-id-accessor/destroy secret_id_accessor=...`.

### SPEC-0084-REQ-013: The cluster needs nothing here

Workloads inside the cluster (the external-secrets operator, SPEC-0083) shall use the store's
Kubernetes auth method and shall not be given AppRoles.

## Degraded behaviour

| Condition | Effect |
|---|---|
| store unreachable | sync fails (exit 1, logged); existing env files stay; exporter keeps running on them |
| login from an unbound address | HTTP 400 `source address "<addr>" unauthorized by CIDR restrictions on the role` (OpenBao appends a harmless `%!w(<nil>)`); nothing changes on the host |
| node deleted and rejoined with a new pod CIDR | logins fail as above; re-bind the role with the new gateway addresses |
| database entry not yet created | S3 env refreshed; `db.env` untouched; message printed |
| audit device cannot write | the store refuses all requests (fail closed); every sync fails until fixed |
| credentials files missing | install does not enable the timer (SPEC-0082 REQ-007) |

What the binding does and does not protect: it stops a leaked `secret_id` being used from any
other LAN machine. It does not stop a compromised pod: anything in the trusted proxy network can
reach the store directly and set its own `X-Forwarded-For`. The `secret_id` is still required
there.

## Verification

1. `sudo systemctl start nms-bao-sync.service; journalctl -u nms-bao-sync -n 5` shows either
   "unchanged" or the files it updated, and `sudo stat -c '%a' /etc/nms-exporter/db.env` is
   `600`.
2. Stability of the source address: run 20 logins from the host and count
   `request.remote_address` for `auth/approle/login` in the audit log: all 20 come from one
   address, the expected gateway address. (In the original, 20/20 came from the node's overlay
   address; a batch of 50 logins on a second node moved that node's address count by exactly 50
   and the first node's by 0.)
3. Forged header: send a login with `X-Forwarded-For: 203.0.113.77` from the host; the audit log
   records the real source, not 203.0.113.77.
4. Wrong host: copy `role_id` and `secret_id` to another LAN machine and log in; it must fail with
   HTTP 400 and the CIDR message above.
5. After a run, the token is gone: the audit log shows `auth/token/revoke-self` right after the
   reads.
6. Change the database password in the store (after `ALTER ROLE`, SPEC-0083 REQ-004), start the
   sync: `db.env` changes and the exporter restarts; run it again: "unchanged", no restart.

## Lessons

See `spec/lessons/SPEC-0099-storage-deploy-lessons.md`: the reverse proxy hides the source
address until the store trusts `X-Forwarded-For` from the proxy's network, a node arrives as its
pod-network gateway address rather than its LAN IP, and `bao unwrap` silently sends your ambient
token.
