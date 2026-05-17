# ADR-019: Traefik Ingress and cert-manager TLS Strategy

## Status

Accepted

## Date

2026-05-15

## Context

AIOStreams (ADR-017) is deployed on K3s and exposed via MetalLB at `http://192.168.1.205:3000`. Stremio on Windows and iOS refuses to install addons served over HTTP — the client silently upgrades manifest URLs from `http://` to `https://`, causing a TLS handshake failure. AIOStreams is functional on macOS where this upgrade does not occur, but the setup is not usable across the household's full device fleet.

K3s was installed with the bundled Traefik ingress controller disabled. No ingress controller or TLS infrastructure exists in the cluster.

Two constraints shape the TLS approach:

- **Internal-only services must not be exposed to the internet.** AIOStreams and any future internal services are intended for LAN access only. Any TLS solution must not require inbound internet connectivity to the cluster.
- **No per-device certificate trust setup.** Manually installing a CA certificate on each device (iOS, Windows, macOS) is operationally unacceptable. New devices must work without device-specific configuration.

These constraints rule out a self-hosted (local) CA: while technically sound, a local CA requires every device to trust the CA certificate before connections succeed. Let's Encrypt is pre-loaded into the OS trust store on every major platform, making it the only CA that satisfies the zero-per-device-setup requirement.

Let's Encrypt requires domain ownership proof via an ACME challenge. The DNS-01 challenge type — where ownership is proved by adding a TXT record to the domain's DNS zone — is the only challenge type that does not require inbound internet access to the cluster (HTTP-01 requires port 80 to be publicly reachable). DNS-01 requires a DNS provider with an API that cert-manager can call to add and remove challenge records. `screamingmorphs.com` is registered at Squarespace, which does not expose a DNS API. Cloudflare offers a free DNS management tier with full API access and first-class cert-manager support.

Publishing a DNS A record for `aiostreams.screamingmorphs.com` pointing to the private IP `192.168.1.205` does not expose the service to the internet. DNS answers a name-to-IP question; routing to RFC-1918 addresses does not exist on the public internet. The only information visible to a public DNS lookup is the private IP address, which reveals nothing useful.

## Decision

**Deploy Traefik as the cluster ingress controller and cert-manager for certificate lifecycle management. Use Let's Encrypt with DNS-01 challenge via Cloudflare for all internal service TLS. Manage DNS for `screamingmorphs.com` via Cloudflare (nameservers delegated from Squarespace); Squarespace remains the domain registrar.**

Internal services are exposed at `<service>.screamingmorphs.com` resolving to their MetalLB LoadBalancer IP. All certificates are issued by Let's Encrypt and renewed automatically by cert-manager before expiry. No inbound internet access to the cluster is required at any point.

## Alternatives Considered

### 1. Local CA (self-signed, cluster-internal)

cert-manager can operate as a self-signed CA entirely within the cluster with no external dependencies.

**Why not chosen:** Every device — including future devices — must have the CA certificate manually installed and trusted before connections succeed. iOS requires a two-step process (install profile, then enable full trust in Settings → About → Certificate Trust Settings). This violates the zero-per-device-setup requirement and creates ongoing operational overhead as new devices are added.

### 2. HTTP-01 ACME Challenge

Let's Encrypt's HTTP-01 challenge type proves domain ownership by serving a file at `http://<domain>/.well-known/acme-challenge/<token>`. This requires Let's Encrypt's servers to reach the cluster on port 80.

**Why not chosen:** Exposing port 80 on the cluster to the public internet is inconsistent with the intranet-only deployment philosophy. DNS-01 achieves the same result without any inbound internet access.

### 3. Keep HTTP, Stremio Workarounds

Stremio on Windows has a flag to disable HTTPS enforcement in some builds. The Stremio web app can sometimes install HTTP addons depending on the platform.

**Why not chosen:** Workarounds are version-dependent and fragile. Stremio iOS enforces HTTPS at the app level with no override. A proper TLS setup is the correct and durable fix.

### 4. Alternative DNS Providers

Route53, DigitalOcean DNS, and others have cert-manager support.

**Why not chosen:** Cloudflare's free DNS tier, API reliability, and cert-manager integration are well-established in the homelab community. `screamingmorphs.com` has no existing Cloudflare dependencies to migrate; the nameserver delegation is a one-time change.

## Consequences

**Positive:**

- All LAN devices connect to internal services over trusted HTTPS with no per-device configuration
- cert-manager renews certificates automatically before expiry (Let's Encrypt certs are 90-day; renewal begins at 60 days)
- Traefik IngressRoute provides a consistent routing layer for all future internal services

**Ongoing operational:**

- **Cloudflare API token:** Stored in OpenBao at `secret/cloudflare/production`, synced into the cluster via ExternalSecret as `cloudflare-api-token`. If the token is rotated, update OpenBao and trigger an ArgoCD sync — cert-manager will pick it up at the next renewal.
- **DNS delegation is permanent:** `screamingmorphs.com` nameservers point to Cloudflare. DNS records for the domain are managed in Cloudflare, not Squarespace. Squarespace manages registration only (renewal billing, WHOIS).
- **Private IPs in public DNS are intentional:** Records like `aiostreams.screamingmorphs.com → 192.168.1.205` are public DNS entries. This is by design and does not constitute internet exposure. Do not add CAA records or other restrictions that would block Let's Encrypt from issuing certs for this zone.
- **New internal services follow this pattern:** Add a DNS A record in Cloudflare pointing to the MetalLB IP, create a Traefik IngressRoute with TLS, and cert-manager issues the cert automatically. No additional cert-manager configuration is required per service.
