# ADR-020: Helm for All K8s Workload Packaging

## Status
Accepted

## Date
2026-05-25

## Context

As the homelab grows to include custom-built applications alongside third-party services,
a consistent packaging standard for K8s workloads is needed. Three options exist:

**Plain K8s YAML manifests** — write `deployment.yaml`, `service.yaml`, etc. directly.
No tooling overhead, but no parameterisation. Running staging and prod requires duplicating
files or accepting that both environments share one manifest — making per-environment
differences (image tag, replicas, resource limits, namespace) impossible to manage cleanly.

**Kustomize** — a `base/` directory holds canonical manifests; `overlays/` apply patches
per environment. Built into `kubectl`, native ArgoCD support, no templating language. The
right middle ground for custom apps with 2–4 environments, but introduces a second
packaging model alongside Helm, which is already in use for third-party apps.

**Helm** — a chart bundles templated manifests with a `values.yaml`. Per-environment
configuration lives in separate values files (`values/staging.yaml`, `values/prod.yaml`).
The dominant standard in corporate environments and across the wider K8s ecosystem. For
third-party apps, upstream maintainers publish charts — operators only provide values.
For custom apps, the chart is authored in the app repository and ArgoCD reads it directly
from git.

The homelab already uses Helm for all third-party apps (ESO, cert-manager, etc.) via
ArgoCD. Introducing Kustomize or plain YAML for custom apps would create two packaging
models to maintain and context-switch between.

## Decision

**All K8s workloads use Helm.**

| App type | Chart source | Values |
|----------|--------------|--------|
| Third-party (ESO, cert-manager, etc.) | Upstream published chart | `values.yaml` per app in infra repo |
| Custom homelab app | Chart authored in app repo, read by ArgoCD from git | `values/staging.yaml`, `values/prod.yaml` in app repo |

ArgoCD Application resources for all workloads live in `tbtaylor100/infra` under
`argocd/apps/`, regardless of where the chart lives. This keeps the GitOps control plane
in one place.

Custom app chart layout:

```
helm/
  <app-name>/
    Chart.yaml
    values.yaml          ← defaults
    values/
      staging.yaml       ← environment overrides
      prod.yaml
    templates/
      deployment.yaml
      service.yaml
      externalsecret.yaml
      configmap.yaml
```

## Alternatives Considered

**Kustomize** — eliminated because it adds a second packaging model to the homelab. Helm
is already in use; keeping one tool reduces cognitive overhead and matches corporate
environment conventions where Helm is the default expectation.

**Plain K8s YAML** — eliminated for multi-environment use. Acceptable only for
single-environment utilities where staging/prod differences are never needed.

## Consequences

- Every new K8s workload, custom or third-party, is packaged as a Helm chart
- Custom app authors must write a chart — a small upfront cost, but the chart is written
  once and thereafter only values files change
- ArgoCD's interface is uniform: every workload is an `Application` pointing at a Helm
  chart with a values file; no mental switching between packaging models
- Skills and patterns transfer directly to corporate environments where Helm is the
  de facto standard
- Plain K8s YAML remains acceptable for one-off debug manifests or single-environment
  utilities that will never need per-environment configuration
