# ADR-010: Forgejo OIDC Tokens for CI Authentication to OpenBao

## Status
Accepted

## Date
2026-04-25

## Context

MAH-69 deployed OpenBao and configured CI (Forgejo Actions) to authenticate via AppRole —
two static credentials (`OPENBAO_ROLE_ID`, `OPENBAO_SECRET_ID`) stored as Forgejo repo
secrets. This was the correct interim choice at the time: the runner was on VM .50, not
a K8s pod, so Kubernetes auth was unavailable and AppRole was the standard machine-auth
pattern.

MAH-82 moved the Forgejo runner to K3s. Forgejo v15 (released 2026-04) ships OIDC
Actions token support (`ACTIONS_ID_TOKEN_REQUEST_URL` + `ACTIONS_ID_TOKEN_REQUEST_TOKEN`),
enabling per-job identity tokens with workflow-level claims (`repository`, `ref`,
`workflow`). The AppRole stored credentials are now a liability with no remaining
technical justification.

The two outstanding problems with the AppRole pattern:

1. **Static credentials** — `OPENBAO_ROLE_ID` and `OPENBAO_SECRET_ID` never expire;
   rotation is manual; anyone with Forgejo repo access can read them.
2. **No per-job scoping** — the same credentials are used by every CI job regardless
   of which workflow or branch triggered it, preventing fine-grained access control.

## Decision

**CI jobs authenticate to OpenBao using per-job Forgejo OIDC tokens. AppRole is
deprecated and will be disabled after verification.**

OpenBao JWT auth backend validates tokens against Forgejo's JWKS endpoint
(`http://192.168.1.50:3000/login/oauth/keys`). Two JWT roles enforce different
access levels:

- **`ci-plan`** — accepts any `ref`, bound to `repository: root/infra`. Used by
  `tofu-plan.yml` (runs on PRs from any branch).
- **`ci-apply`** — restricted to `ref: refs/heads/main`. Used by `tofu-apply.yml`.
  OpenBao rejects the JWT if the `ref` claim is anything other than `refs/heads/main`,
  making it impossible to run `tofu apply` from a PR branch even if someone tries.

**External Secrets Operator (ESO) is also deployed as foundational K8s infrastructure.**

ESO is not used for CI workflows (which use per-job OIDC). It provides the standard
pattern for future K8s application pods to pull secrets from OpenBao without any stored
credentials. ESO's ServiceAccount authenticates to OpenBao via Kubernetes auth
(`eso-reader` role). Deployed now so the pattern is available without rework when
needed.

Key configuration choices:

- **Forgejo v15 upgrade** — required for OIDC token endpoint; handled by new Ansible
  role on VM .50
- **JWKS validation** — OpenBao fetches Forgejo's public keys dynamically; token
  validation is cryptographic, not shared-secret
- **Ansible `uri` module for OpenBao config** — runs from localhost against the API
  directly; no `kubectl exec` shell-quoting fragility; idempotent and re-runnable
- **`ClusterSecretStore` scoped cluster-wide** — any future namespace can reference it
  without duplicating connection config

## Consequences

- **No stored credentials anywhere in Forgejo** — the two AppRole secrets are deleted
  after verification. Future credential compromise vectors are eliminated from the
  CI path.
- **Per-job token expiry** — OIDC tokens are issued at job start and expire when the
  job ends. Leaked tokens cannot be replayed.
- **Branch-level `tofu apply` enforcement** — `ci-apply` role rejects any JWT without
  `ref: refs/heads/main`. This is enforced by OpenBao, not by workflow conditions —
  it cannot be bypassed by editing the workflow YAML.
- **Forgejo OIDC is ephemeral by design** — `ACTIONS_ID_TOKEN_REQUEST_URL` is only
  valid during an active job; hitting it from outside a runner returns 404. This is
  correct security behavior, not a bug.
- **ESO is foundational, not yet functional** — CRDs and `ClusterSecretStore` are
  live; `ExternalSecret` resources for specific apps will be created as those apps
  are built. ESO adds no operational burden until it's used.
- **Forgejo v15 is a hard dependency** — OIDC token support does not exist in v14;
  the Ansible upgrade playbook must run before the JWT auth backend is configured.
- **OpenBao must remain unsealed** — unchanged from ADR-005; sealed OpenBao causes
  CI failures with a clear HTTP 503 error.
- **AppRole auth is disabled after verification** — `bao auth disable approle`.
  Re-enabling requires the OpenBao root token.
