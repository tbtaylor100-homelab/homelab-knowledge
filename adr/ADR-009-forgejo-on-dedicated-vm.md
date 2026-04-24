# ADR-009: Forgejo Runs on a Dedicated VM Outside K3s

## Status
Accepted

## Date
2026-04-24

## Context

Forgejo is the Git server for the homelab. Every piece of infrastructure depends on it:

- **ArgoCD** continuously polls Forgejo for changes to K3s application manifests
- **Forgejo Actions** runs CI pipelines (OpenTofu plan/apply, image builds) triggered by PRs
- **Ansible playbooks and OpenTofu configs** are stored in Forgejo repos — the authoritative source for rebuilding the cluster

The question arose whether Forgejo should move to K3s to be consistent with other services (managed by ArgoCD, Helm chart, standard observability pipeline).

## Decision

**Forgejo remains on a dedicated VM (VM 102, `192.168.1.50`) outside K3s. It must never move into K3s.**

The reason is a bootstrapping/circular dependency problem:

- K3s workloads are defined in Forgejo repos and deployed by ArgoCD
- ArgoCD runs inside K3s and polls Forgejo to reconcile cluster state
- If Forgejo runs inside K3s and K3s goes down, Forgejo goes down with it
- To rebuild K3s, you need ArgoCD configs — which live in Forgejo — which is down
- There is no way out of this loop without Forgejo being available independently

With Forgejo on a dedicated VM, the recovery path is always available:

1. K3s node crashes
2. SSH into VM 102 — Forgejo is still up
3. Pull configs from Forgejo, re-run `tofu apply` and `ansible-playbook`
4. K3s comes back; ArgoCD reconnects to Forgejo and reconciles

The consistency benefit of running Forgejo in K3s does not outweigh the risk of losing the recovery mechanism entirely.

## Consequences

- Forgejo is not managed by ArgoCD — it is a Docker Compose deployment on VM 102, outside the GitOps pipeline it serves
- VM 102 must be treated as a critical, independently maintained host — its own backups, its own uptime monitoring (Uptime Kuma checks `192.168.1.50`)
- Ansible cannot currently target VM 102 (no SSH access as of 2026-04-24); any Forgejo-level changes require console access
- Log shipping for Forgejo (VM 102) must use the Ansible Promtail role rather than the in-cluster Promtail DaemonSet, since VM 102 is not a K3s node
- Future services that act as sources of truth for cluster recovery (e.g., a secrets manager) should follow the same pattern — evaluate whether they belong inside or outside K3s based on whether their loss would prevent cluster recovery
