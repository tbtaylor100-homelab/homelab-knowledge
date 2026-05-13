# ADR-017: Deploy AIOStreams on K3s as a Stremio Filtering Proxy

## Status

Accepted

## Date

2026-05-13

## Context

AIOStreams is a self-hosted addon aggregator for Stremio that consolidates multiple debrid providers and addon sources into a single filtering proxy, removing blocked or undesirable content from search results before they are displayed in the Stremio client.

On or around **2026-05-10**, Real-Debrid implemented content filtering on their platform, blocking cached torrents identified by filename keywords including `WEB-DL`, `WEBRip`, `AMZN`, `NF`, `YTS`, `RARBG`, and `EZTV`. Real-Debrid returns `infringing_file` responses instead of playable links for these tagged files. This is a platform-level policy change, not an account ban or temporary outage.

The impact on the homelab Stremio experience was immediate: 50%+ of Torrentio search results became dead links. Every search for popular content showed WEB-DL and YTS entries that appeared in the UI but failed on click with "infringing file" errors. The Stremio user experience degraded to the point where the setup was functionally unusable for new releases.

A filtering proxy is the appropriate solution: AIOStreams intercepts addon responses (from Torrentio and other sources) before they reach the Stremio client and removes filtered results based on configurable regex patterns. Only whitelisted, playable streams appear.

Two constraints shape this deployment:

- **`SECRET_KEY` is immutable.** Generated once during initial provisioning, this key encrypts user configuration data in AIOStreams' SQLite database. If changed, all saved user configurations are invalidated and must be re-entered. It is stored in OpenBao and synced via ExternalSecret; it is never rotated.
- **Intranet-only exposure.** AIOStreams is reachable at `http://192.168.1.205:3000` via MetalLB LoadBalancer. No Traefik IngressRoute or public DNS entry is configured. External internet access is not a requirement for this homelab deployment.

## Decision

**Deploy self-hosted AIOStreams v2.29.5 on K3s, managed by ArgoCD, to filter Real-Debrid-blocked content from Stremio search results before they surface to the user.**

AIOStreams is classified as a K3s application per ADR-016: it is user-facing, no other service depends on it, and it runs on top of the platform rather than enabling it. The deployment follows established homelab patterns: manifests in `kubernetes/aiostreams/` watched by ArgoCD, credentials stored in OpenBao at `secret/aiostreams/production` and synced into the cluster via ExternalSecret Operator.

## Alternatives Considered

### 1. Comet addon

Comet is an alternative filtering addon for Stremio with Real-Debrid support, designed as a drop-in replacement for Torrentio with built-in debrid caching.

**Why not chosen:** Comet is less mature than AIOStreams with a smaller community. It has fewer debrid provider integrations and more limited regex filtering capability compared to AIOStreams' `WHITELISTED_REGEX_PATTERNS` system. AIOStreams has an established self-hosting path and active development; Comet's self-hosted path requires more configuration overhead for similar functionality.

### 2. ElfHosted hosted AIOStreams

ElfHosted offers a managed SaaS deployment of AIOStreams on their infrastructure, eliminating the need for self-hosting.

**Why not chosen:** ElfHosted hosted instances sometimes disable Torrentio for licensing reasons, removing a key content source. Self-hosting preserves full Torrentio access and intranet isolation — the AIOStreams instance is reachable only from LAN devices, not exposed to the public internet. Additionally, ElfHosted adds an ongoing subscription cost for a single-user homelab where the operator already has K3s infrastructure available.

### 3. Switch debrid providers (TorBox, Usenet)

Migrating from Real-Debrid to a provider that has not implemented the May 2026 content blocking (e.g., TorBox or a Usenet-based workflow).

**Why not chosen:** Provider migration requires rotating API keys and updating workflow configuration across all Stremio addons. TorBox is less tested in this homelab environment. Usenet has slower upload performance for new releases and requires a different client stack. Real-Debrid with AIOStreams filtering is the most direct path to restoring the existing setup without a full provider migration, and it preserves the existing Real-Debrid investment.

## Consequences

**Positive:**

- Stremio search results no longer show dead-link WEB-DL, AMZN, YTS, RARBG, and EZTV entries
- AIOStreams UI provides centralized control over debrid providers, addon sources, and regex filter patterns
- GitOps-managed deployment (ArgoCD + ExternalSecret) follows established homelab patterns from ADR-016

**Ongoing operational:**

- **Quarterly regex review:** `WHITELISTED_REGEX_PATTERNS` in `kubernetes/aiostreams/configmap.yaml` must be reviewed quarterly as Real-Debrid's block list evolves. Update the ConfigMap and trigger an ArgoCD sync (`argocd app sync aiostreams`) to apply changes without a pod restart.
- **`SECRET_KEY` immutability:** Generated once during Phase 1 provisioning via `openssl rand -hex 32`, stored in OpenBao at `secret/aiostreams/production`, and synced via ExternalSecret. Never rotated. If the OpenBao secret is lost, a new deployment requires wiping the SQLite database at `/app/data/db.sqlite` and re-onboarding all user configurations.
- **Intranet-only exposure is an accepted constraint:** AIOStreams is reachable at `http://192.168.1.205:3000` on LAN only. External internet access would require a Traefik `IngressRoute` and internal DNS (deferred to PLAT-01/PLAT-02 milestones).
- **SQLite persistence is adequate for single-user homelab:** The SQLite database at `/app/data` on a `local-path` PVC is sufficient for the current single-user setup. Multi-replica HA would require PostgreSQL (deferred to RES-02).
