---
id: SPEC-0085
title: 'Exposing the map on the LAN'
description: 'A friendly LAN-only name for the map: the cluster''s reverse proxy routes it to the exporter port on the gaming PC through a selectorless Service and a hand-applied EndpointSlice, with a LAN DNS override.'
type: snapshot
area: storage-deploy
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
depends_on: [SPEC-0082]
supersedes: []
superseded_by: []
related_specs: [SPEC-0081, SPEC-0099, SPEC-0060, SPEC-0070]
screenshots: []
---

# SPEC-0085: Exposing the map on the LAN

## Summary

The map and systems pages (SPEC 0060–0079) are served by the exporter itself on the gaming PC,
port 9877, because only that machine can read the save and the hook's files. For a phone or
tablet on the LAN to reach it by a stable HTTPS name, the home cluster's reverse proxy (Traefik
in the original) fronts it: a Kubernetes `Service` with no selector, an `EndpointSlice` pointing
at the gaming PC's LAN address and port, and an `Ingress` with the LAN wildcard certificate. A
LAN DNS override points the name at the reverse proxy's load-balancer address. The name is
LAN-only and has no single sign-on in front: it exposes nothing that `<gaming PC>:9877` does not
already expose, and the map polls `/api/live` every 0.5 s with no session, which an SSO redirect
would break.

A rebuilder without a cluster can skip all of this and open `http://<gaming PC>:9877/map`, or put
any reverse proxy (Caddy, nginx) in front. The exporter needs nothing special: a bare `/` from a
browser (Accept contains `text/html`) is redirected 302 to `/map`, so the bare hostname works.

## Inputs and outputs

| Item | Value |
|---|---|
| backend | `<gaming PC LAN IP>:9877`, plain HTTP |
| name | `<map name>` under the LAN domain, e.g. `nms-map.<lan domain>` |
| TLS | the LAN wildcard certificate already used by the proxy |
| DNS | a host override in the LAN resolver: `<map name>` -> the reverse proxy's LB address |

## Requirements

### SPEC-0085-REQ-001: Selectorless Service

The cluster shall have a `Service` with no `selector` and one port `http`/9877/TCP, in the
namespace the proxy's edge backends live in.

### SPEC-0085-REQ-002: Manual EndpointSlice

The cluster shall have an `EndpointSlice` labelled `kubernetes.io/service-name: <service>`,
`addressType: IPv4`, port `http`/9877/TCP, one endpoint with the gaming PC's LAN address and
`conditions.ready: true`:

```yaml
apiVersion: v1
kind: Service
metadata: {name: nms-map, namespace: <edge ns>}
spec:
  ports: [{name: http, port: 9877, protocol: TCP}]
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: nms-map
  namespace: <edge ns>
  labels: {kubernetes.io/service-name: nms-map}
addressType: IPv4
ports: [{name: http, port: 9877, protocol: TCP}]
endpoints:
  - addresses: ["<gaming PC LAN IP>"]
    conditions: {ready: true}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: nms-map, namespace: <edge ns>}
spec:
  ingressClassName: traefik
  tls: [{hosts: [<map name>], secretName: <wildcard tls secret>}]
  rules:
    - host: <map name>
      http:
        paths:
          - {path: /, pathType: Prefix, backend: {service: {name: nms-map, port: {number: 9877}}}}
```

### SPEC-0085-REQ-003: EndpointSlices applied outside GitOps

Where the cluster is managed by Argo CD with its default resource exclusions, the EndpointSlice
shall be applied imperatively (`kubectl apply` of just the EndpointSlice documents from the
manifest file), and re-applied after any change to the backend address or port. Argo's default
`resource.exclusions` silently drop `discovery.k8s.io/EndpointSlice`: the application shows
Synced while the slice never exists, and the Ingress answers 503 "no ready endpoints". Keep the
slice in the same file as the Service for review; un-excluding the kind globally would make Argo
watch every controller-managed slice in the cluster.

### SPEC-0085-REQ-004: Ingress

The cluster shall have an `Ingress` for `<map name>` with the LAN wildcard TLS secret, path `/`
prefix, to the Service on 9877, with no authentication middleware.

### SPEC-0085-REQ-005: DNS override

The LAN resolver shall resolve `<map name>` to the reverse proxy's load-balancer address (a host
override). No public DNS record shall exist for it.

### SPEC-0085-REQ-006: LAN only

The name shall not be reachable from outside the LAN: no port forward, no public DNS, no tunnel.
The exporter has no authentication and accepts `POST /api/finds/class`.

### SPEC-0085-REQ-007: Exporter listens on all interfaces

The exporter shall listen on `0.0.0.0:9877` (SPEC-0082) so that both the proxy and the metrics
scraper can reach it. The host firewall, if any, shall allow 9877 from the LAN.

### SPEC-0085-REQ-008: Without a cluster

Where there is no cluster, any LAN reverse proxy shall do the same job: terminate TLS for
`<map name>` with a LAN certificate and proxy everything to `http://<gaming PC>:9877`, with no
authentication, buffering off for the 0.5 s poll, and the DNS override pointing at that proxy.
For example (Caddy):

```
<map name> {
    tls /etc/ssl/lan/fullchain.pem /etc/ssl/lan/key.pem
    reverse_proxy <gaming PC LAN IP>:9877
}
```

or nginx:

```nginx
server {
    listen 443 ssl;
    server_name <map name>;
    ssl_certificate     /etc/ssl/lan/fullchain.pem;
    ssl_certificate_key /etc/ssl/lan/key.pem;
    location / {
        proxy_pass http://<gaming PC LAN IP>:9877;
        proxy_buffering off;
        proxy_set_header Host $host;
    }
}
```

The exporter does not use `Host` or `X-Forwarded-*`, and its pages use root-absolute paths
(`/api/...`, `/icon/...`, `/glyph/...`), so it works behind any name but not under a sub-path:
proxy the whole name to it.

## Degraded behaviour

| Condition | Effect |
|---|---|
| gaming PC off or exporter stopped | proxy answers 502/504 for the name; nothing else affected |
| EndpointSlice missing (dropped by GitOps, or never applied) | proxy answers 503 "no ready endpoints" while the Service looks correct; re-apply the slice |
| gaming PC's LAN address changes | same as missing; reserve the address in DHCP |
| database down | the map loads and shows its "Database down" banner (SPEC-0081) |
| GitOps app shows Synced on an old commit after a push | a hard refresh and a manual sync of the app were needed once; check the app's revision, not just its status |

## Verification

1. `dig +short <map name>` from a LAN client returns the proxy's LB address.
2. `kubectl -n <edge ns> get endpointslice nms-map -o wide` lists the gaming PC's address and
   9877. If it is missing, the GitOps tool dropped it: apply it by hand.
3. `curl -sI https://<map name>/` returns `302` with `Location: /map` when sent
   `-H 'Accept: text/html'`; `curl -s https://<map name>/healthz` prints `ok`.
4. `curl -s https://<map name>/api/live | head -c 200` returns JSON (or the 503 body of
   SPEC-0081 if the database is down, which also proves the route reaches the exporter).
5. From outside the LAN (phone on mobile data) the name does not resolve.

## Lessons

See `spec/lessons/SPEC-0099-storage-deploy-lessons.md`: Argo CD silently drops EndpointSlices,
and an app can sit "Synced" on an old commit after a push.
