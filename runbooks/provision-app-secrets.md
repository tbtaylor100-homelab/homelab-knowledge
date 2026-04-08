# Runbook: Provision Secrets for a New Application

Use this runbook when onboarding a new application that needs its own isolated
credentials per environment. Each app+environment combination gets a dedicated:

- KV path: `secret/<app>/<env>`
- Policy: `<app>-<env>-policy` (read-only, scoped to that path only)
- AppRole: `<app>-<env>` (authenticated via role_id + secret_id)

This means a staging credential breach cannot expose production secrets, and one
app cannot read another app's secrets.

## Path & Naming Convention

```
secret/
  homelab/
    ci/          ← shared CI infra credentials (Proxmox, MinIO, etc.)
  <app>/
    staging/     ← all staging credentials for <app>
    production/  ← all production credentials for <app>
```

## Prerequisites

- OpenBao is unsealed and reachable at `http://192.168.1.210:8200`
- You have the root token (or a token with `auth/` and `policy/` write access)
- AppRole auth method is already enabled (it is, from the CI setup)

---

## Step 1 — Write the initial credentials

Create the KV path and populate it with the app's secrets. Replace `<app>`, `<env>`,
and the key/value pairs with real values.

```bash
bao kv put -address=http://192.168.1.210:8200 secret/<app>/<env> \
  DB_PASSWORD=changeme \
  API_KEY=changeme
```

> Use `kv patch` for subsequent additions to avoid overwriting existing keys.

Verify:

```bash
bao kv get -address=http://192.168.1.210:8200 secret/<app>/<env>
```

---

## Step 2 — Create a scoped policy

The policy grants read-only access to exactly one path. Save it to a temp file
then write it to OpenBao:

```bash
cat > /tmp/<app>-<env>-policy.hcl << 'EOF'
path "secret/data/<app>/<env>" {
  capabilities = ["read"]
}
EOF
bao policy write -address=http://192.168.1.210:8200 <app>-<env>-policy /tmp/<app>-<env>-policy.hcl
```

Verify:

```bash
bao policy read -address=http://192.168.1.210:8200 <app>-<env>-policy
```

---

## Step 3 — Create the AppRole

```bash
bao write -address=http://192.168.1.210:8200 auth/approle/role/<app>-<env> \
  token_policies=<app>-<env>-policy \
  token_ttl=1h \
  token_max_ttl=4h
```

---

## Step 4 — Retrieve the role_id and secret_id

```bash
bao read -address=http://192.168.1.210:8200 auth/approle/role/<app>-<env>/role-id
```

```bash
bao write -address=http://192.168.1.210:8200 -f auth/approle/role/<app>-<env>/secret-id
```

> The secret_id is shown only once. Store it immediately in Forgejo (Step 5).
> To rotate, generate a new secret_id and update the Forgejo secret.

---

## Step 5 — Store in Forgejo as repository secrets

In the app's Forgejo repository, add two repository secrets:

| Secret name | Value |
|-------------|-------|
| `OPENBAO_ROLE_ID` | role_id from Step 4 |
| `OPENBAO_SECRET_ID` | secret_id from Step 4 |

> If multiple apps share a single Forgejo org, consider org-level secrets scoped
> to specific repositories to avoid per-repo duplication.

---

## Step 6 — Read credentials in a CI workflow

```yaml
- name: Fetch secrets from OpenBao
  env:
    OPENBAO_ADDR: http://192.168.1.210:8200
  run: |
    TOKEN=$(curl -s -X POST $OPENBAO_ADDR/v1/auth/approle/login \
      -d "{\"role_id\":\"$OPENBAO_ROLE_ID\",\"secret_id\":\"$OPENBAO_SECRET_ID\"}" \
      | jq -r '.auth.client_token')
    SECRETS=$(curl -s -H "X-Vault-Token: $TOKEN" \
      $OPENBAO_ADDR/v1/secret/data/<app>/<env>)
    echo "DB_PASSWORD=$(echo $SECRETS | jq -r '.data.data.DB_PASSWORD')" >> $GITHUB_ENV
    echo "API_KEY=$(echo $SECRETS | jq -r '.data.data.API_KEY')" >> $GITHUB_ENV
```

---

## Rotating a secret_id

Secret IDs should be rotated periodically or after a suspected leak:

```bash
bao write -address=http://192.168.1.210:8200 -f auth/approle/role/<app>-<env>/secret-id
```

Update `OPENBAO_SECRET_ID` in Forgejo with the new value. The old secret_id is
automatically invalidated.

---

## Checklist summary

- [ ] `secret/<app>/<env>` KV path created and populated
- [ ] `<app>-<env>-policy` written and verified
- [ ] `<app>-<env>` AppRole created with that policy
- [ ] role_id and secret_id stored in Forgejo repository secrets
- [ ] CI workflow reads and injects secrets successfully
