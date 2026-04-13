# ADR-008: Forgejo Actions Runner on K3s with Docker-in-Docker

## Status
Accepted

## Date
2026-04-13

## Context

Forgejo CI workflows build and push Docker images to the Forgejo container registry
(`192.168.1.50:3000`, plain HTTP). The Forgejo host is inaccessible — no SSH access,
no Proxmox console — so TLS cannot be configured natively on it. Without a working
runner, no CI jobs execute.

Three options were considered:

1. **Enable TLS on Forgejo directly** — generate a self-signed CA + server cert, configure
   Forgejo's Docker Compose to use it, distribute the CA to all clients. Blocked by lack
   of host access.

2. **K3s reverse proxy for TLS termination** — deploy cert-manager + nginx on K3s to proxy
   HTTPS traffic to `http://192.168.1.50:3000`. Requires a new MetalLB IP and changes to
   all client configurations. Adds infrastructure complexity for a service that is
   internal-only and already working over HTTP.

3. **Move the runner to K3s with insecure-registry** — deploy the Forgejo Actions runner
   as a K8s Deployment on K3s (192.168.1.60), configure Docker-in-Docker (DinD) with
   `--insecure-registry=192.168.1.50:3000`. No changes to Forgejo host, no changes to
   existing client configurations.

## Decision

**Deploy the Forgejo Actions runner as a K8s Deployment with a DinD sidecar.**

The runner pod has:
- **runner container**: `code.forgejo.org/forgejo/runner` polling Forgejo for jobs
- **dind sidecar**: `docker:27-dind` (privileged) providing a Docker daemon configured
  with `--insecure-registry=192.168.1.50:3000`

This isolates the insecure-registry concern to the DinD sidecar inside the runner pod —
it does not affect the K3s node's containerd configuration or any other workload. The
runner is managed by ArgoCD (GitOps) and its registration token is stored as a K8s secret
via Ansible, consistent with all other homelab secrets management.

The runner registers with Forgejo using the label `ubuntu-latest`, matching the existing
`runs-on: ubuntu-latest` in `repowise-image.yml` — no workflow changes required.

## Consequences

- CI workflows work without any changes to Forgejo's host configuration
- The insecure-registry flag is scoped to DinD inside the runner pod; no system-wide
  Docker daemon or containerd changes needed on K3s node
- The runner pod requires `securityContext.privileged: true` for the DinD sidecar — an
  accepted trade-off for a single-node homelab
- The `.runner` registration file lives in an emptyDir volume; pod restarts trigger
  re-registration with Forgejo (old offline runner entries accumulate in Forgejo admin,
  but do not affect functionality)
- If the Forgejo host becomes accessible in the future, enabling native TLS would allow
  removing the insecure-registry flag and potentially retiring the DinD sidecar in favor
  of a lighter runner setup
