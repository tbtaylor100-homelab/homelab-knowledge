# ADR-016: Infrastructure Services Run on Proxmox VMs; Applications Run on K3s

## Status
Accepted

## Date
2026-05-12

## Context

The homelab has a clear tooling hierarchy for managing resources:

- **OpenTofu** — provisions Proxmox VMs
- **Ansible** — installs and configures software on VMs (including the K3s node itself)
- **ArgoCD** — manages workloads running inside K3s

As the number of services grows, the question of *where a new service should run* arises repeatedly. Running everything on K3s is operationally convenient (ArgoCD, Helm, standard observability) but creates circular dependencies when the service is something K3s itself depends on. Running everything on dedicated VMs avoids that problem but adds provisioning overhead for services that don't need it.

The pattern from prior decisions (ADR-009: Forgejo on a dedicated VM) established that services which K3s depends on must not run inside K3s. This ADR generalises that into a durable decision rule applicable to all future services.

## Decision

**Infrastructure services run on Proxmox VMs, managed by OpenTofu and Ansible. Applications run on K3s, managed by ArgoCD.**

The classification criterion is:

> **If the service going down would prevent K3s from being rebuilt, or if it serves K3s rather than running on top of it, it is infrastructure and belongs on a Proxmox VM.**

### Clear infrastructure — Proxmox VM

| Service | Why |
|---------|-----|
| Forgejo | Source of truth for all configs; ArgoCD cannot function without it (ADR-009) |
| OpenBao | Secrets backend for ESO, CI pipelines, and all apps; K3s workloads depend on it at startup |
| DNS / internal resolver | K3s nodes and VMs need name resolution to function at all |
| NFS / shared storage backend | Block/file storage used by multiple K3s PVs — loss affects cluster-wide PV availability |
| Monitoring backend (e.g. Grafana, Loki, Prometheus) | Debatable — see grey area below |

### Clear applications — K3s

| Service | Why |
|---------|-----|
| AIOStreams | User-facing app; no other service depends on it |
| ESO (External Secrets Operator) | K3s-native controller; depends on OpenBao, does not enable it |
| ArgoCD | K3s-native GitOps controller |
| Any app built on this homelab | Depends on the platform, is not part of the platform |

### Grey area — databases

Databases do not fit neatly into either category. The placement depends on scope:

**Shared / platform database → Proxmox VM**
A Postgres or MySQL instance used by multiple applications, or by platform tooling (e.g. Gitea, a metrics store, an auth service), is infrastructure. Loss affects multiple tenants. Manage with OpenTofu + Ansible.

**App-specific database → K3s is acceptable**
A database that exists solely to serve one K3s application (e.g. a SQLite PVC, or a Postgres instance deployed by that app's Helm chart) can colocate with the application on K3s. It lives and dies with the app, which is the expected behaviour. If the app is later decomissioned the database goes with it.

The deciding question: *does anything outside this application depend on this database?* If yes, it is infrastructure. If no, it can stay on K3s.

### Grey area — observability stack

Monitoring and logging tools (Prometheus, Grafana, Loki, Promtail) are a special case. They observe K3s, but K3s does not depend on them — an outage does not prevent recovery. They are acceptable on K3s for operational consistency, with the understanding that observability is unavailable precisely when you most need it (during a K3s outage). Moving them to VMs buys visibility during cluster-level incidents at significant provisioning cost. Treat as a future decision based on operational pain.

## Consequences

- Any new service must be explicitly classified before provisioning. The default question is: *does anything outside K3s depend on this?*
- Infrastructure services (Proxmox VMs) are not managed by ArgoCD. They have their own provisioning runbooks in `homelab-knowledge/runbooks/`.
- Application services (K3s) are not managed by Ansible after initial node setup. Changes go through GitOps (PR → ArgoCD sync).
- Services that are currently misclassified (e.g. OpenBao running on K3s) should be migrated. See MAH-94.
- This decision intentionally accepts that the observability stack may be blind during K3s outages. That is a known, accepted trade-off, not an oversight.
