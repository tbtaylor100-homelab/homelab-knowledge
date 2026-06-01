# ADR-024: Traefik LoadBalancer externalTrafficPolicy: Local

## Status

Accepted

## Date

2026-05-31

## Context

Traefik is deployed as the cluster ingress controller (ADR-019) with a MetalLB LoadBalancer service at `192.168.1.206`. Staging services require IP-based access control — specifically, `staging.screamingmorphs.com` must be restricted to LAN clients (`192.168.1.0/24`) to prevent accidental public exposure in the event of a router misconfiguration.

Traefik's `ipAllowList` middleware enforces this restriction by checking the source IP of incoming requests. However, Kubernetes' default `externalTrafficPolicy: Cluster` causes kube-proxy to apply SNAT (Source Network Address Translation) to all ingress packets before they reach the Traefik pod. kube-proxy rewrites the real client source IP (e.g. `192.168.1.187`) with a cluster-internal IP (e.g. `10.42.x.x`). Traefik receives the SNAT'd address and evaluates middleware against it — causing the `ipAllowList` to block all traffic, including legitimate LAN clients.

`externalTrafficPolicy: Cluster` exists to support cross-node load balancing: when a packet arrives at a node that does not run the target pod, kube-proxy forwards it cross-node. To ensure the response travels back through the same node (maintaining routing symmetry), kube-proxy SNAT's the source address. This is necessary in multi-node clusters with sparse pod placement.

The homelab K3s cluster currently runs on a single node. Cross-node forwarding does not occur. The SNAT provides no benefit in this topology while actively breaking IP-based middleware.

## Decision

**Set `externalTrafficPolicy: Local` on the Traefik LoadBalancer service via `HelmChartConfig`.**

With `Local` policy, kube-proxy does not perform SNAT. Packets are delivered to the Traefik pod with the original client source IP intact. IP-based middleware (`ipAllowList`, rate limiting, access logging) operates against real client addresses.

The `HelmChartConfig` in `infra/kubernetes/traefik/helm-chart-config.yaml` sets:

```yaml
service:
  spec:
    externalTrafficPolicy: Local
```

## Alternatives Considered

### 1. Keep `externalTrafficPolicy: Cluster`, remove IP allowlist middleware

The private IP `192.168.1.206` is physically unreachable from the public internet (RFC-1918). The allowlist was defence-in-depth only.

**Why not chosen:** Removing the allowlist eliminates a useful safety layer. If port 443 is ever accidentally forwarded on the router, staging would be publicly accessible with no application-layer guard. Defence-in-depth is worth preserving when the fix is low-cost.

### 2. Keep `externalTrafficPolicy: Cluster`, update allowlist to cluster CIDR

With SNAT in place, Traefik sees all traffic as originating from a cluster-internal IP (`10.42.x.x`). The allowlist could be updated to permit `10.42.0.0/16` (the K3s pod CIDR) instead of `192.168.1.0/24`.

**Why not chosen:** This makes the allowlist security theatre. Because kube-proxy rewrites *every* incoming packet's source to a cluster-internal IP — regardless of where it originated — the allowlist ends up permitting all traffic that reaches `192.168.1.206`. A LAN client and an attacker who has gained LAN access are indistinguishable to Traefik: both arrive with a `10.42.x.x` source after SNAT. The allowlist no longer filters by who is connecting; it filters by a network address that is always true. Defence-in-depth requires the check to actually discriminate between allowed and disallowed clients.

### 3. Configure Traefik `proxyProtocol` with MetalLB

MetalLB's L2 mode does not support PROXY protocol. L7 PROXY protocol requires an L4 load balancer in front of Traefik (e.g. HAProxy, Nginx stream mode) — significant additional infrastructure for a single-node homelab.

**Why not chosen:** Disproportionate complexity. `externalTrafficPolicy: Local` achieves the same result with a one-line config change.

### 4. Use `X-Forwarded-For` header inspection

Traefik can be configured to trust and extract real IPs from `X-Forwarded-For` headers set by an upstream proxy.

**Why not chosen:** There is no upstream proxy in this topology. `X-Forwarded-For` headers from clients are untrustworthy and trivially spoofed. This approach would defeat the purpose of the allowlist.

## Consequences

**Positive:**

- Real client IPs are visible to all Traefik middleware — allowlists, rate limiting, and access logs reflect actual client addresses
- Staging IP restriction works as intended
- No cross-node SNAT overhead (marginal performance gain on single-node)

**Operational (multi-node future):**

- With `Local` policy, MetalLB only announces the LoadBalancer IP from nodes that have a healthy Traefik pod. If Traefik does not run on a node (e.g. via taint/toleration), that node will not receive traffic for this service. This is correct behaviour but must be considered when scaling the cluster — ensure Traefik runs on all nodes expected to receive external traffic, or accept that traffic concentrates on nodes with Traefik pods.
- If a second node is added and Traefik does not run on it, that node will be silently excluded from MetalLB's announcement rotation. Monitor MetalLB speaker logs when adding nodes.

**Related decisions:**

- ADR-019: Traefik and cert-manager TLS strategy
- ADR-023: OpenBao secret path convention (staging/production environment secrets)
