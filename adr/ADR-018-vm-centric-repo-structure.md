# ADR-018: VM-Centric Monorepo Structure for Infra Repository

## Status
Accepted

## Date
2026-05-13

## Context

The `infra` repository is organised by tool: `opentofu/` contains VM provisioning code, `ansible/` contains configuration management, `kubernetes/` contains workload manifests. This mirrors how the tools themselves are structured, but not how an operator thinks about the homelab.

When answering "how is k3s set up?", a reader must know to look in three separate directories — `opentofu/stacks/k3s/` for the VM definition, `ansible/playbooks/k3s.yml` for software installation, and `ansible/roles/k3s/` for the role logic — none of which point to each other. The relationship between files is implicit, requiring deep familiarity with both tools and the repo layout.

As the homelab grows, this friction compounds: adding a new VM means touching multiple unrelated directories with no single place that tells the full story of that machine.

### Considered alternatives

**Split repos** — separate `infra` (OpenTofu + Ansible) and `gitops` (Kubernetes + ArgoCD) repositories. Rejected at current scale: added coordination overhead, cross-repo secrets duplication, and ArgoCD reconfiguration outweigh the benefit for a single-operator homelab. Revisit if the Kubernetes workload surface grows significantly or multiple operators are involved.

**READMEs as navigation** — leave files in place and add per-VM `README.md` index files. Rejected as the primary approach: READMEs go stale and do not reduce the actual navigation burden; they add a maintenance surface without solving the root cause.

## Decision

**Reorganise the `infra` repository around VMs rather than tools.**

Each VM gets a top-level directory under `vms/` containing everything needed to understand and operate that machine:

```
vms/
  k3s-control-01/
    README.md          ← what this VM is, IP, role in the homelab
    provision/         ← OpenTofu stack (was opentofu/stacks/k3s/)
    k3s.yml            ← Ansible playbook (was ansible/playbooks/k3s.yml)
  uptime-kuma-01/
    README.md
    provision/
    uptime-kuma.yml
  homelab-01/
    README.md
    forgejo-upgrade.yml
ansible/
  inventory/           ← stays central; applies across all VMs
  roles/               ← stays central; Ansible tooling expects this layout
  ansible.cfg
opentofu/
  modules/             ← stays central; shared by all VM stacks
kubernetes/            ← stays; ArgoCD watches this path on main
argocd/
```

### What moves

| From | To |
|------|----|
| `opentofu/stacks/k3s/` | `vms/k3s-control-01/provision/` |
| `opentofu/stacks/uptime-kuma/` | `vms/uptime-kuma-01/provision/` |
| `ansible/playbooks/k3s.yml` | `vms/k3s-control-01/k3s.yml` |
| `ansible/playbooks/uptime-kuma.yml` | `vms/uptime-kuma-01/uptime-kuma.yml` |
| `ansible/playbooks/argocd.yml` | `vms/k3s-control-01/argocd.yml` |
| `ansible/playbooks/forgejo-upgrade.yml` | `vms/homelab-01/forgejo-upgrade.yml` |
| `ansible/playbooks/mcp-servers.yml` | `vms/k3s-control-01/mcp-servers.yml` |
| `ansible/playbooks/openbao-auth-config.yml` | `vms/homelab-01/openbao-auth-config.yml` |

### What stays

- `ansible/inventory/` — inventory and group_vars apply across VMs; central location avoids duplication
- `ansible/roles/` — Ansible role discovery requires a consistent `roles_path`; moving roles into per-VM dirs requires complex multi-path `ansible.cfg` configuration with no meaningful benefit
- `opentofu/modules/proxmox-vm/` — shared module used by all VM stacks; belongs in a central location
- `kubernetes/` — ArgoCD is configured to watch this path; moving it requires ArgoCD reconfiguration (separate decision)

### Required follow-up changes

- Update `opentofu/modules/proxmox-vm` relative path in each moved stack (`source = "../../../opentofu/modules/proxmox-vm"`)
- Update `ansible.cfg` `playbook_dir` or document that playbooks are run from their VM directory
- Update any CI pipeline paths that reference the old locations (`.forgejo/workflows/`)

## Consequences

- A reader can navigate to `vms/k3s-control-01/` and immediately see both the provisioning definition and the configuration playbooks for that machine.
- Ansible roles and inventory remain in `ansible/`, which is a minor inconsistency but required by tooling conventions.
- Kubernetes manifests remain in `kubernetes/`, consistent with ADR-016 (infra on VMs, apps on K3s) — workloads are not VM-specific.
- OpenTofu module paths in each stack must be updated after the move.
- This structure scales linearly: adding a new VM means creating a new `vms/<name>/` directory with no changes to existing VM directories.
