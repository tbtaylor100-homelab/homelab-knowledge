# ADR-022: CD Tag Writeback via Forgejo Contents API

## Status
Accepted

## Date
2026-05-26

## Context

The `screaming-morphs` CD pipeline builds a Docker image on every push to `main`,
then writes the new image tag back to `helm/screaming-morphs/values/staging.yaml`
so ArgoCD can reconcile the staging deployment. The writeback commit must land on
`main`, which is a protected branch (push allowlist: `tbtaylor100` only).

The original implementation (`feat: add Dockerfile, Helm chart, and CI pipeline`,
PR #9) used a git commit + push approach:

```bash
git config --global user.name "github-actions[bot]"
git remote set-url origin http://${{ github.actor }}:${{ secrets.FORGEJO_TOKEN }}@...
git push origin main
```

This worked initially but relied on `secrets.FORGEJO_TOKEN` being the auto-generated
Forgejo Actions job token. That token represents the Forgejo Actions bot identity,
not `tbtaylor100`. The bot is not in the push allowlist, so the push returned 403.

After switching registry auth to OIDC (ADR-010, `ci-registry` role), the PAT stored
in OpenBao at `secret/homelab/ci` was updated to include `repo:write` scope alongside
`registry:write`. Using this PAT for the git push resolved the 403, but Forgejo's
pre-receive hook began returning `Internal Server Error` regardless of auth method,
token format (URL-embedded vs HTTP header), or user identity. The HTTP layer returns
200; the error is encoded inside the git protocol response body. The server-side cause
could not be determined — Forgejo logs were inaccessible without direct shell access
to the host VM, and the container name was not `forgejo`.

The Forgejo Contents API (`PUT /api/v1/repos/:owner/:repo/contents/:path`) was tested
as an alternative. It succeeded immediately and has continued to work reliably. It
operates outside the git receive-pack path and is not subject to pre-receive hooks.

## Decision

**CD tag writebacks use the Forgejo Contents API, not git push.**

The token used is `FORGEJO_REGISTRY_TOKEN`, fetched from OpenBao at job start via
OIDC (`ci-registry` role, ADR-010). The same token handles docker registry auth and
the Contents API write — both require `registry:write` and `repo:write` scope on the
PAT stored at `secret/homelab/ci`.

The writeback step:

```bash
FILE_INFO=$(curl -f -H "Authorization: token $FORGEJO_REGISTRY_TOKEN" "$API")
FILE_SHA=$(echo "$FILE_INFO" | jq -r '.sha')
NEW_CONTENT=$(echo "$FILE_INFO" | jq -r '.content' | base64 -d \
  | sed 's/tag: .*/tag: "main-<sha>"/' | base64 -w 0)
curl -f -X PUT "$API" \
  -H "Authorization: token $FORGEJO_REGISTRY_TOKEN" \
  -d "{\"message\":\"chore: update image tag [skip ci]\",\"content\":\"$NEW_CONTENT\",\"sha\":\"$FILE_SHA\",\"branch\":\"main\"}"
```

The `sha` field in the PUT body is required by Forgejo to prevent concurrent write
conflicts — it must match the current file SHA returned by the GET.

## Consequences

- **Git push to protected `main` is not used from CI** — the pre-receive hook
  errors internally. This is a known Forgejo server-side issue on this instance
  that has not been investigated further.
- **Contents API commits appear as the PAT owner** — commits will show as
  `tbtaylor100`, matching the PAT identity. This is correct and consistent.
- **`set -e` is required in the writeback step** — without it, a failed GET curl
  silently passes an empty `FILE_SHA` to the PUT, which returns 422. `set -e`
  ensures the step fails fast at the GET if auth or the file path is wrong.
- **Token scope dependency** — the PAT at `secret/homelab/ci` must retain both
  `registry:write` and `repo:write`. Rotating to a read-only token will silently
  break the writeback.
- **`[skip ci]` in the commit message** — prevents CI from re-running on the tag
  writeback commit. Required status checks are configured on `main`; without this
  flag, a CI run would be triggered that passes, but it is unnecessary.
