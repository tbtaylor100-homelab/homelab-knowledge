# ADR-017: Deploy AIOStreams on K3s as a Stremio Addon Aggregator

## Status

Accepted

## Date

2026-05-13

## Context

AIOStreams is a self-hosted addon aggregator for Stremio that consolidates multiple content providers and addon sources into a single configurable proxy. It filters stream results based on source quality and availability before they are displayed in the Stremio client, removing entries that would fail on playback.

The homelab Stremio setup depends on multiple addon sources that can return unplayable stream links. A filtering proxy is the appropriate solution: AIOStreams intercepts addon responses before they reach the Stremio client and removes streams based on configurable regex patterns. Only accessible streams appear in the UI.

Two constraints shape this deployment:

- **`SECRET_KEY` is immutable.** Generated once during initial provisioning, this key encrypts user configuration data in AIOStreams' SQLite database. If changed, all saved user configurations are invalidated and must be re-entered. It is stored in OpenBao and synced via ExternalSecret; it is never rotated.
- **Intranet-only exposure.** AIOStreams is reachable on the LAN via MetalLB LoadBalancer. External internet access is not a requirement for this homelab deployment.

## Decision

**Deploy self-hosted AIOStreams v2.29.5 on K3s, managed by ArgoCD, to filter unavailable streams from Stremio search results before they surface to the user.**

AIOStreams is classified as a K3s application per ADR-016: it is user-facing, no other service depends on it, and it runs on top of the platform rather than enabling it. The deployment follows established homelab patterns: manifests in `kubernetes/aiostreams/` watched by ArgoCD, credentials stored in OpenBao at `secret/aiostreams/production` and synced into the cluster via ExternalSecret Operator.

## Alternatives Considered

### 1. Comet addon

Comet is an alternative Stremio addon with similar content aggregation and filtering capabilities.

**Why not chosen:** Comet is less mature than AIOStreams with a smaller community. It has fewer provider integrations and more limited regex filtering capability compared to AIOStreams' `WHITELISTED_REGEX_PATTERNS` system. AIOStreams has an established self-hosting path and active development; Comet's self-hosted path requires more configuration overhead for similar functionality.

### 2. Managed hosted AIOStreams

Managed hosting services offer AIOStreams as a SaaS deployment, eliminating the need for self-hosting.

**Why not chosen:** Managed instances sometimes restrict addon sources for operational reasons, removing key content sources. Self-hosting preserves full addon access and intranet isolation — the AIOStreams instance is reachable only from LAN devices, not exposed to the public internet. Additionally, managed hosting adds an ongoing subscription cost for a single-user homelab where the operator already has K3s infrastructure available.

### 3. Alternative provider migration

Migrating to a different content provider or protocol stack.

**Why not chosen:** Provider migration requires rotating credentials and updating configuration across all Stremio addons. Filtering at the AIOStreams layer is the most direct path to a reliable setup without a full provider migration.

## Consequences

**Positive:**

- Stremio search results no longer show dead-link or inaccessible stream entries
- AIOStreams UI provides centralized control over providers, addon sources, and regex filter patterns
- GitOps-managed deployment (ArgoCD + ExternalSecret) follows established homelab patterns from ADR-016

**Ongoing operational:**

- **Periodic regex review:** `WHITELISTED_REGEX_PATTERNS` in `kubernetes/aiostreams/configmap.yaml` must be reviewed periodically as stream availability patterns evolve. Update the ConfigMap and trigger an ArgoCD sync (`argocd app sync aiostreams`) to apply changes without a pod restart.
- **`SECRET_KEY` immutability:** Generated once during Phase 1 provisioning via `openssl rand -hex 32`, stored in OpenBao at `secret/aiostreams/production`, and synced via ExternalSecret. Never rotated. If the OpenBao secret is lost, a new deployment requires wiping the SQLite database at `/app/data/db.sqlite` and re-onboarding all user configurations.
- **Intranet-only exposure is an accepted constraint:** AIOStreams is reachable on LAN only. External internet access would require a Traefik `IngressRoute` and internal DNS (deferred).
- **SQLite persistence is adequate for single-user homelab:** The SQLite database at `/app/data` on a `local-path` PVC is sufficient for the current single-user setup. Multi-replica HA would require PostgreSQL (deferred).
