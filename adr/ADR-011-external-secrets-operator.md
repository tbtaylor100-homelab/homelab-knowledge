# ADR-011: External Secrets Operator for K8s Secret Management

## Status
Accepted

## Date
2026-04-25

## Context

K8s application pods need access to secrets stored in OpenBao (the homelab secrets manager — see ADR-005). Without a standard pattern, each application would need to implement its own secret-fetching logic: either a custom init container, a sidecar, or baking credentials into the pod spec. This creates inconsistency and couples application manifests to the secrets backend.

The CI runner (Forgejo Actions) has its own distinct secret access pattern using per-job Forgejo OIDC tokens (see ADR-010) and does not use ESO.

## Decision

Deploy External Secrets Operator (ESO) to K3s as the standard mechanism for K8s applications to consume OpenBao secrets.

ESO is deployed via ArgoCD (`argocd/apps/external-secrets.yaml`) and configured with a cluster-scoped `ClusterSecretStore` that authenticates to OpenBao using the Kubernetes auth backend. Application teams declare an `ExternalSecret` resource in their namespace; ESO reads from OpenBao and writes a native K8s `Secret`. The application pod consumes the K8s Secret normally via `envFrom` or volume mount.

**Authentication:** ESO authenticates to OpenBao using its K8s ServiceAccount token via the Kubernetes auth backend. The `eso-reader` role in OpenBao binds the `external-secrets` ServiceAccount in the `external-secrets` namespace to the `eso-policy` (read access on `secret/data/homelab/ci`).

**Scope:** `ClusterSecretStore` is cluster-scoped so any namespace can reference it without duplicating connection config.

## Alternatives Considered

**Vault Agent Sidecar** — injects a sidecar into each pod to fetch and refresh secrets. More complex per-pod setup; requires annotations on every deployment. ESO's operator pattern is less invasive.

**Direct OpenBao API calls in app init containers** — each app fetches its own secrets at startup. Duplicates auth logic across every application. No standardization.

**K8s Secrets committed to git** — plaintext or SOPS-encrypted secrets in the repo. Introduces credential sprawl and a separate encryption key management problem. OpenBao is already the established secrets backend.

## Consequences

- Applications declare secrets as `ExternalSecret` resources pointing at `ClusterSecretStore/openbao` — no OpenBao client code in application containers
- Adding a new secret to an application requires: (1) write the secret to OpenBao, (2) declare an `ExternalSecret` — no redeployment of ESO
- ESO must be healthy for application secrets to sync; a liveness check on `ClusterSecretStore` status surfaces problems early
- CI workflows (Forgejo Actions) do **not** use ESO — they use per-job OIDC tokens directly (ADR-010). ESO is for long-running K8s pods only.
