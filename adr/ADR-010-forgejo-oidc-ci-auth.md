# ADR-010: Forgejo OIDC Tokens for CI Authentication to OpenBao

## Status
Accepted

## Date
2026-04-25

## Context

MAH-69 deployed OpenBao and configured CI (Forgejo Actions) to authenticate via AppRole.
The AppRole `role_id` and `secret_id` were hardcoded as static Forgejo repo secrets
(`OPENBAO_ROLE_ID`, `OPENBAO_SECRET_ID`) — written once into Forgejo's secret store,
never rotated, and used identically by every workflow and every job regardless of branch,
trigger, or intent. This is a hardcoded credential pattern: the values are fixed at
configuration time, stored indefinitely, and indistinguishable from secrets baked into
code. The only thing separating them from plaintext in the repo is Forgejo's encryption
at rest. This was the accepted interim choice: the runner was on VM .50, Kubernetes auth
was unavailable, and AppRole was the standard machine-auth option at the time.

MAH-82 moved the Forgejo runner to K3s. Forgejo v15 (released 2026-04) ships OIDC
Actions token support (`ACTIONS_ID_TOKEN_REQUEST_URL` + `ACTIONS_ID_TOKEN_REQUEST_TOKEN`),
enabling per-job identity tokens with workflow-level claims (`repository`, `ref`,
`workflow`). The hardcoded AppRole credentials are now a liability with no remaining
technical justification.

The two outstanding problems with the AppRole pattern:

1. **Hardcoded static credentials** — `OPENBAO_ROLE_ID` and `OPENBAO_SECRET_ID` never
   expire; rotation is manual and disruptive; any workflow in the repo — including a
   malicious one on a PR branch — can use them to authenticate to OpenBao with the
   same level of access as a trusted main-branch deploy.
2. **No per-job scoping** — the same credentials are used by every CI job regardless
   of which workflow or branch triggered it, preventing fine-grained access control.

## Decision

**CI jobs authenticate to OpenBao using per-job Forgejo OIDC tokens. AppRole is
deprecated and will be disabled after verification.**

OpenBao JWT auth backend is configured with `oidc_discovery_url` pointing at Forgejo's
Actions OIDC issuer (`http://192.168.1.50:3000/api/actions`). OpenBao auto-discovers the
JWKS URI from the discovery document, so token validation is cryptographic and the JWKS
URL is not hardcoded in config. Two JWT roles enforce different access levels:

- **`ci-plan`** — accepts any `ref`, bound to `repository: tbtaylor100/infra`. Used by
  `tofu-plan.yml` (runs on PRs from any branch).
- **`ci-apply`** — restricted to `ref: refs/heads/main`. Used by `tofu-apply.yml`.
  OpenBao rejects the JWT if the `ref` claim is anything other than `refs/heads/main`,
  making it impossible to run `tofu apply` from a PR branch even if someone tries.

K8s application pods (not CI) use a separate pattern — External Secrets Operator with
the Kubernetes auth backend — covered in ADR-011.

Key configuration choices:

- **Forgejo v15 upgrade** — required for OIDC token endpoint; handled by new Ansible
  role on VM .50
- **OIDC discovery** — `oidc_discovery_url` points at Forgejo's Actions issuer root;
  OpenBao fetches the JWKS URI from `/.well-known/openid-configuration` automatically;
  token validation is cryptographic, not shared-secret
- **Ansible `uri` module for OpenBao config** — runs from localhost against the API
  directly; no `kubectl exec` shell-quoting fragility; idempotent and re-runnable

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
- **Forgejo v15 is a hard dependency** — OIDC token support does not exist in v14;
  the Ansible upgrade playbook must run before the JWT auth backend is configured.
- **OpenBao must remain unsealed** — unchanged from ADR-005; sealed OpenBao causes
  CI failures with a clear HTTP 503 error.
- **AppRole auth is disabled after verification** — `bao auth disable approle`.
  Re-enabling requires the OpenBao root token.
