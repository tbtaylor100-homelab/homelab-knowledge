# ADR-004: OpenTofu Manages VM Resources, Ansible Manages VM Configuration

## Status
Accepted

## Date
2026-04-04

## Context

During early homelab bootstrapping, the `proxmox-vm` OpenTofu module used a
cloud-init `vendor_data` snippet to install `qemu-guest-agent` on first boot.
This was a workaround: cloud-init was the only mechanism available at VM
creation time before Ansible was wired into the provisioning workflow.

This approach caused two problems:

1. **Layer violation** — OpenTofu was reaching inside the guest OS to install
   software. That is configuration management, not infrastructure provisioning.

2. **Destroy+recreate cascade** — the bpg/proxmox provider treats
   `source_raw.data` as a force-new field. Any change to the snippet content
   destroys and recreates the file resource, which cascades to VM recreation
   because the VM stores the file ID in its cloud-init config. Recreating a
   Proxmox VM assigns a new random MAC address, breaking router DHCP
   reservations and any services that depend on a stable IP.

## Decision

**OpenTofu is responsible for VM resources. Ansible is responsible for
everything inside the VM.**

| Concern | Tool |
|---|---|
| VM exists with correct CPU, RAM, disk, network | OpenTofu |
| Static IP, SSH key seeded at creation | OpenTofu |
| Packages installed, services enabled, OS hardened | Ansible |

Cloud-init `vendor_data` must not be used in OpenTofu modules. It is an
OS-level configuration mechanism delivered through an infrastructure channel —
a workaround that belongs in Ansible.

The correct provisioning order is:
1. `tofu apply` — VM is created with hardware config and SSH key
2. `ansible-playbook` — OS is configured (packages, services, hardening)

## Consequences

- The `proxmox-vm` module has no `vendor_data` resource
- `qemu-guest-agent` and any other guest OS packages are installed via the
  Ansible `base` role
- Any future need to run something on first boot must be implemented as an
  Ansible task, not a cloud-init snippet
- There is a brief window between `tofu apply` completing and Ansible running
  where `qemu-guest-agent` is not present — this is acceptable
- See `infra/docs/vm-provisioning.md` for the operational runbook
