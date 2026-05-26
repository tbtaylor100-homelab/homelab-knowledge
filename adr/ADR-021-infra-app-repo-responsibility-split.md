# ADR-021: Infra Repo Owns Secrets and CD; App Repo Owns Application Configuration

## Status
Accepted

## Date
2026-05-25

## Context

The homelab uses two separate git repositories for application workloads:

- **`tbtaylor100/infra`** — manages infrastructure: K3s manifests, ArgoCD Applications,
  secrets management, namespace definitions
- **`tbtaylor100/<app>`** — manages the application itself: source code, Dockerfile,
  Helm chart, values files

As custom applications join the homelab (starting with `screaming-morphs`), a clear
ownership boundary is needed to answer: *which repo controls which resources?*

Without an explicit rule, responsibilities drift. App developers might add ExternalSecret
templates to their Helm charts (coupling the app to OpenBao path structure). Infra
changes might require app repo PRs. Secret access could be escalated by an app developer
without an infra review.

The two repos are intentionally coupled — the infra repo deploys and configures the app
repo's artifacts — but the direction of that coupling must be one-way: the infra repo
references the app repo, not the reverse.

## Decision

**The infra repo owns secrets management and CD mechanics. The app repo owns application
configuration.**

### Infra repo owns

| Resource | Location | Why |
|----------|----------|-----|
| `ExternalSecret` | `kubernetes/<app>/` | Controls which secrets from OpenBao the app can access; app should not define its own secret access |
| `Namespace` | `kubernetes/<app>/` | Namespace creation is an infra operation |
| ArgoCD `Application` | `argocd/apps/` | CD pipeline mechanics (sync policy, destination, target revision) are an infra decision |

### App repo owns

| Resource | Location | Why |
|----------|----------|-----|
| Helm chart | `helm/<app>/templates/` | App owner controls how their app is structured |
| Values files | `helm/<app>/values/staging.yaml`, `values/prod.yaml` | App owner controls replicas, resource limits, image tags, app-level env config |
| `Dockerfile` | repo root | App owner controls the image build |
| CI workflow | `.forgejo/workflows/` | App owner controls how the image is built and tagged |

### The boundary in practice

The app's Helm chart references the K8s Secret by name (e.g. `screaming-morphs-secret`)
without knowing or controlling how it was created. ESO creates it in the app's namespace
from OpenBao. The app consumes it via `envFrom` or `env.valueFrom` in the Deployment
template. Adding or changing a secret requires an infra repo PR — not an app repo change.

```
Infra repo                         App repo
──────────────────────────────     ──────────────────────────────
ExternalSecret (OpenBao → K8s)  →  Deployment consumes K8s Secret by name
ArgoCD Application              →  Helm chart + values files
Namespace                       →  (none — namespace is infra concern)
```

### Directory structure (infra repo, per app)

```
kubernetes/<app>/
  staging/
    namespace.yaml          ← sync-wave: "-2"
    external-secret.yaml    ← sync-wave: "-1", namespace: <app>-staging
  production/
    namespace.yaml
    external-secret.yaml
```

Two ArgoCD Applications manage the infra side — one per environment — each pointing at
the relevant staging or production subdirectory.

## Alternatives Considered

**Everything in the app repo (including ExternalSecrets)** — rejected. An app developer
could modify their own ExternalSecret to access secrets outside their intended scope.
Secrets access must require an infra PR with a separate review gate.

**Everything in the infra repo (including Helm chart and values)** — rejected. App
configuration changes (replicas, resource limits, image tags) would require infra repo
PRs, coupling app development velocity to infra review. App owners should control how
their app runs.

**Single monorepo** — not a fit for this homelab's tooling split (OpenTofu/Ansible for
infra, Forgejo Actions for app CI). ADR-016 already establishes the Proxmox/K3s boundary;
the same principle applies at the repo level.

## Consequences

- ExternalSecret templates are never added to Helm charts. If an app's Helm chart
  contains an ExternalSecret, it is a policy violation.
- Adding a new secret *key* to an app does not require an infra repo PR. Each app's
  ExternalSecret maps to the app's dedicated OpenBao path using `dataFrom` (extract all
  properties); adding a key to OpenBao at that path makes it available in the K8s Secret
  automatically on the next refresh interval.
- Changing *which* OpenBao path an app has access to requires an infra repo PR. This is
  the access escalation gate — an app cannot grant itself access to a new secrets path.
- App developers can iterate on replicas, resource limits, and image config without
  touching the infra repo.
- The infra repo's `kubernetes/<app>/` directory uses per-environment subdirectories
  (`staging/`, `production/`) to allow separate ArgoCD Applications per environment,
  each with its own destination namespace.
- The coupling between repos is one-directional: the infra repo's ArgoCD Application
  references the app repo's Helm chart. The app repo has no reference to the infra repo.
