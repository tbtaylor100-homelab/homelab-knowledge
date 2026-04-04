# ADR-005: OpenBao as Self-Hosted Secrets Manager

## Status
Accepted

## Date
2026-04-04

## Context

Credentials (Proxmox API token, MinIO access keys, SSH public key) were managed as
gitignored files on the operator's Mac. This made CI impossible: the Forgejo Actions
runner on VM .50 has no access to the operator's local files, so `tofu-plan.yml`
could not execute `tofu plan` on PRs. There was also no mechanism for secret rotation,
audit logging, or scoping credentials to specific consumers.

Forgejo repository secrets were considered as an interim solution but rejected:
they offer no per-path access control (a compromise of one secret exposes all), no
built-in rotation workflow, and no audit trail. They are appropriate for storing
pointers to a secrets manager (e.g. an AppRole secret ID), not the secrets themselves.

HashiCorp Vault was the original reference point (cited in `group_vars/nas/secrets.yml.example`)
but its licence changed to BUSL-1.1 in 2023, making it unsuitable for a self-hosted
homelab that may involve community use. OpenBao is its MPL-2.0 fork, maintaining
full API compatibility while remaining genuinely open source.

## Decision

**OpenBao is the secrets manager for this homelab. It runs as a single-node K3s
workload, deployed via ArgoCD using the official OpenBao Helm chart.**

Key configuration choices:

- **Single-node standalone mode** — HA is not needed for a homelab; Raft storage
  can be adopted later if replica count grows
- **File storage backend** — simplest durable option; no external dependency
- **TLS disabled** — traffic stays on the homelab LAN; enable if exposure changes
- **LoadBalancer service at `192.168.1.210`** (MetalLB pool) — provides a stable,
  network-accessible endpoint for the runner on VM .50 and for the operator CLI
- **AppRole auth for CI** — the runner on .50 is not a K8s pod, so Kubernetes auth
  is not available; AppRole is the standard Vault pattern for non-cloud machine auth;
  only two values (`role_id` + `secret_id`) live in Forgejo, replacing five raw secrets
- **Token auth for local operator** — root token from `bao operator init`, stored in
  a password manager; rotate to a named userpass account in a follow-on
- **Manual unseal** — sealed state on restart is the correct default; auto-unseal is
  a follow-on concern

Secret structure uses KV v2 at `secret/homelab/ci` for all CI-required credentials.
The CI policy grants read-only access to this single path.

## Consequences

- The runner's `tofu-plan.yml` and `tofu-apply.yml` workflows now fetch all infra
  credentials from OpenBao via AppRole login; only `OPENBAO_ROLE_ID` and
  `OPENBAO_SECRET_ID` remain as Forgejo repo secrets
- OpenBao must be unsealed after every pod restart — operator must run
  `bao operator unseal` before CI jobs can succeed; a sealed OpenBao causes
  workflow failures with a clear error (HTTP 503 from the health endpoint)
- Rotating credentials is now a `bao kv put` command, not a file edit on the Mac
- All secret access is logged in OpenBao's audit log (enable with
  `bao audit enable file file_path=/vault/logs/audit.log`)
- The Vault Agent Injector is disabled — if future workloads need secrets injected
  as files or env vars into K8s pods, enable it or adopt the External Secrets Operator
- Moving the runner to K3s (follow-on) will allow switching to Kubernetes auth,
  eliminating the two remaining Forgejo secrets entirely
- MinIO root password in `group_vars/nas/secrets.yml` and Ansible `secrets.yml`
  files remain as gitignored local files — migration to OpenBao is a follow-on
