---
id: SPEC-0092
title: 'Address-bound, non-expiring machine roles for host secrets'
description: 'Why bare-metal hosts get AppRole-style roles with secret IDs that never expire, bound to the host''s source address, instead of periodic tokens or long-lived credentials in files.'
type: decision
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: accepted
supersedes: []
superseded_by: []
related_specs: [SPEC-0084, SPEC-0083]
---

# SPEC-0092: Address-bound, non-expiring machine roles for host secrets

## Context

Two bare-metal hosts in the original needed secrets unattended: the gaming PC (save-backup S3
key, database login) and a workstation running certificate-refresh jobs. Inside the Kubernetes
cluster this is solved: workloads use the secret store's Kubernetes auth. Outside it, the options
all involve a credential sitting on the host.

Before this decision the workstation's certificate jobs used a **periodic token**: a token that
lives as long as it is renewed within its period. It worked until the host was off through one
renewal window; the token expired, and from then on the jobs failed until a person logged in and
minted a new one. A credential that dies when the machine sleeps is wrong for a machine that is
allowed to sleep.

The owner's requirement was that these hosts, and the cluster, "should be able to access secrets
like that without authentication", meaning without a person in the loop, while a stolen copy of
the credential is not enough on its own.

## Decision

Each host gets its own AppRole-style role in the secret store (OpenBao/Vault) with:

- `secret_id_ttl=0`, `secret_id_num_uses=0`: the secret ID never expires and can be reused. The
  host can be off for any length of time.
- `secret_id_bound_cidrs` and `token_bound_cidrs` set to the `/32` addresses the store sees
  the host as. A copied secret ID is useless from anywhere else.
- `token_ttl=900`, `token_max_ttl=3600`, and every job revokes its token when done (`revoke-self`
  on exit). There is no long-lived token anywhere.
- A policy granting `read` on only the entries the host needs.
- The secret ID minted once by an administrator, response-wrapped, and unwrapped on the host into
  a root-only file.
- An audit device on the store so every machine read is recorded, secrets HMAC'd.

Because the store sits behind a reverse proxy, the store trusts `X-Forwarded-For` from the
proxy's network only, and the binding uses the address a host actually arrives as. For a host
that is also a cluster node, that is its pod-network gateway address, not its LAN IP (SPEC-0084).

## Alternatives considered

- **Periodic tokens.** No address binding needed, but they die after one missed renewal. This is
  the failure that prompted the change.
- **Long-TTL tokens (months).** Same failure on a longer clock, and a stolen token works from
  anywhere until it expires.
- **Expiring secret IDs with rotation.** More secure in the abstract, but rotation needs something
  that is already authenticated, which is the problem being solved. It moves the "a person must
  log in" moment, it does not remove it.
- **Secrets in files placed by hand, no store.** No moving parts, but no rotation path, no audit,
  and the value lives in two places that drift.
- **TLS client certificates (cert auth method).** A good fit in principle and binds identity to a
  key rather than an address. Rejected for now: needs a CA and certificate lifecycle on each
  host, i.e. another expiring credential, and the proxy in front would terminate TLS, so the
  store would never see the client certificate without passthrough.
- **Make the hosts in-cluster workloads.** Not possible: the exporter must run on the gaming PC
  to read the save.

## Consequences

- A host that is off for weeks still syncs at the next boot.
- The binding depends on network topology. If the proxy changes, if a node rejoins with a new pod
  CIDR, or if a host stops being a cluster node, logins fail with a clear CIDR error and the
  role must be re-bound. That is documented with the command to do it.
- Non-node hosts cannot use this binding as is: their requests arrive as whichever node the load
  balancer announces from, which moves on failover. They need either a dedicated LB IP with
  `externalTrafficPolicy: Local` or another auth method.
- The binding does not protect against a compromised pod inside the trusted proxy network, which
  can reach the store directly and set its own `X-Forwarded-For`. The secret ID is still needed
  there, so the binding is a second factor, not the only one.
- The audit log is now load-bearing: audit devices fail closed, so a full log partition on the
  store's node stops every secret read.
- Every job on this pattern has the same three steps (log in, read, revoke on exit), which makes
  new ones easy to review.

Revisit if the store gains a better machine identity that survives a proxy (e.g. mTLS with
passthrough), or if the network is restructured so source addresses are no longer stable.
