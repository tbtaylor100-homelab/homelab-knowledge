# ADR-023: Hierarchical OpenBao Secret Path Convention

## Status

Accepted

## Date

2026-05-31

## Context

As custom applications join the homelab, each app stores secrets in OpenBao and syncs them into Kubernetes via External Secrets Operator (ADR-011). The first app (`screaming-morphs`) initially used flat paths:

```
screaming-morphs/staging
screaming-morphs/production
```

Each path held all secrets for that environment in a single KV entry, synced via `dataFrom: extract` into one K8s Secret.

This breaks down when an app needs more than one category of secrets per environment. For `screaming-morphs`, adding a Cloudflare Tunnel credential alongside app runtime secrets creates a conflict: `dataFrom: extract` pulls every key from the path into the app Secret, meaning the tunnel credential would leak into the app pod's environment variables. Splitting secrets into separate categories requires separate paths.

A consistent naming convention is needed that:
- Scales to multiple secret categories per app and environment
- Is readable and unambiguous
- Applies uniformly to all current and future apps

## Decision

**Use a three-level hierarchical path: `<app>/<env>/<purpose>`**

```
<app>/staging/environment
<app>/staging/<other-purpose>
<app>/production/environment
<app>/production/<other-purpose>
```

### Segment definitions

| Segment | Values | Notes |
|---------|--------|-------|
| `<app>` | kebab-case app name (e.g. `screaming-morphs`) | Matches the app's repo and K8s namespace prefix |
| `<env>` | `staging`, `production` | Matches the environment subdirectory in the infra repo |
| `<purpose>` | `environment`, `cloudflare-tunnel`, etc. | Describes what the secret is used for, not what it contains |

### The `environment` purpose

App runtime secrets (database URLs, API keys, feature flags, etc.) always live at `<app>/<env>/environment`. This is the path the standard ExternalSecret template references. Naming it `environment` reflects that these values are injected as environment variables into the app container.

### Other purposes

Additional secret categories get their own path under the same `<app>/<env>/` prefix. Each maps to a separate ExternalSecret and a separate K8s Secret, keeping secret access scoped to the consumer that needs it.

Example for `screaming-morphs`:

```
screaming-morphs/staging/environment       → app env vars (staging)
screaming-morphs/production/environment    → app env vars (production)
screaming-morphs/production/cloudflare-tunnel  → tunnel credential JSON
```

## Migration

Existing flat paths (`screaming-morphs/staging`, `screaming-morphs/production`) must be migrated to the new convention. Migration steps:

1. Write the same secret values to the new paths in OpenBao
2. Update the ExternalSecret `key:` references in the infra repo
3. Verify ArgoCD syncs the updated ExternalSecrets successfully
4. Delete the old flat paths from OpenBao

## Alternatives Considered

**Flat paths with suffixes** (e.g. `screaming-morphs/production-tunnel`)
Avoids nesting but is harder to read and doesn't group by environment cleanly. Rejected.

**Single path per environment with all purposes merged**
Simplest, but forces `dataFrom: extract` to pull all secrets into one K8s Secret, leaking unrelated credentials into app pods. Rejected.

**Purpose-first paths** (e.g. `cloudflare-tunnel/screaming-morphs/production`)
Organises by infra tool rather than by app. Makes it harder to audit what secrets a given app has access to. Rejected.

## Consequences

- All new apps must use `<app>/<env>/<purpose>` paths from the start.
- The standard ExternalSecret template references `<app>/<env>/environment`.
- Adding a new secret category (e.g. a webhook signing key) requires a new OpenBao path and a new ExternalSecret — it does not extend the existing `environment` path.
- Changing which OpenBao path an app has access to still requires an infra repo PR (per ADR-021).

## References

- ADR-005: OpenBao Secrets Manager
- ADR-011: External Secrets Operator
- ADR-021: Infra Repo Owns Secrets and CD; App Repo Owns Application Configuration
