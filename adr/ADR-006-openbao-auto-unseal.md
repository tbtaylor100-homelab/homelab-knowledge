# ADR-006: OpenBao Auto-Unseal via K8s Secret + Sidecar

## Status
Accepted

## Date
2026-04-08

## Context

ADR-005 deployed OpenBao in standalone mode with manual unseal, noting: "sealed state on
restart is the correct default; auto-unseal is a follow-on concern." That follow-on is
now addressed here.

The consequence of manual unseal: any OpenBao pod restart (K3s node drain, pod eviction,
OOM kill, ArgoCD sync that triggers a rollout) causes all CI jobs to fail with HTTP 503
until an operator runs `bao operator unseal`. This creates an unacceptable operational
burden and makes the homelab brittle.

Four auto-unseal approaches were evaluated:

| Approach | Verdict | Reason |
|----------|---------|--------|
| Transit auto-unseal | Rejected | Requires a second OpenBao instance to unseal the first — circular dependency |
| Cloud KMS (AWS/GCP) | Rejected | Adds external dependency; homelab has no existing cloud footprint |
| Age/SOPS sealed secret | Rejected | Requires SOPS controller or sidecar tooling; no benefit over K8s Secret for on-cluster use |
| K8s Secret + sidecar | **Accepted** | Pragmatic, self-contained, unseal key protected by K3s etcd encryption at rest |

## Decision

**The unseal key is stored in a K8s Secret (`openbao-unseal-key`, namespace `openbao`).
A sidecar container polls OpenBao's health endpoint and calls `POST /v1/sys/unseal`
automatically whenever the pod starts sealed.**

Key implementation choices:

- **K8s Secret, not git** — the unseal key is never committed to the `infra` repo; the
  Secret is created once by the operator via `kubectl create secret` and is not managed
  by ArgoCD
- **Sidecar, not init container** — init containers run before the main container starts,
  so OpenBao is not yet listening; a sidecar runs in parallel and can poll the health
  endpoint until OpenBao is ready
- **REST API, not `bao` CLI** — `POST /v1/sys/unseal` is called directly via curl from
  a `curlimages/curl` image, avoiding image coupling with the OpenBao release
- **Poll interval 30 s** — the sidecar stays alive and re-checks every 30 seconds,
  handling edge cases where OpenBao re-seals after an operator-initiated seal

The `extraContainers` and `extraVolumes` fields in the OpenBao Helm chart
(`infra/argocd/apps/openbao.yaml`) carry this configuration, keeping it GitOps-managed.

The K8s Secret is created once by the operator:

```bash
kubectl create secret generic openbao-unseal-key \
  --namespace openbao \
  --from-literal=key=<UNSEAL_KEY>
```

## Consequences

- OpenBao pod restarts no longer require operator intervention — CI jobs recover
  automatically within ~35 seconds (30 s poll + 5 s startup wait)
- The unseal key lives on-cluster inside etcd, which K3s encrypts at rest; the threat
  model is equivalent to any other sensitive K8s Secret in the cluster
- Key rotation: update the K8s Secret (`kubectl create secret --dry-run=client -o yaml |
  kubectl apply -f -`), then delete the OpenBao pod to force a re-seal/re-unseal cycle
  with the new key
- The sidecar adds a second container to the OpenBao pod; `kubectl get pods -n openbao`
  will show `2/2 Running` instead of `1/1`
- If the unseal key is lost, OpenBao cannot be auto-unsealed — the key must be re-entered
  manually via `bao operator unseal` and the Secret recreated
- ADR-005 is superseded on the "manual unseal" point; all other ADR-005 decisions remain
  in effect
