# Runbook: Add a New Credential

Use this runbook any time a new secret (API token, password, access key) needs to be
available to CI workflows or other homelab consumers.

## Overview

Credentials live in OpenBao at `secret/homelab/ci`. CI jobs authenticate via AppRole
and read from that path. The only two values stored in Forgejo are `OPENBAO_ROLE_ID`
and `OPENBAO_SECRET_ID` — everything else is fetched at runtime.

## Prerequisites

- OpenBao is unsealed and reachable at `http://192.168.1.210:8200`
- You have the root token (or a token with write access to `secret/homelab/ci`)

## Steps

### 1. Write the credential to OpenBao

Add the new key to the existing KV secret. Replace `MY_NEW_KEY` and `my-value`:

```bash
bao kv patch -address=http://192.168.1.210:8200 secret/homelab/ci MY_NEW_KEY=my-value
```

> Use `kv patch` (not `kv put`) to add a key without overwriting existing ones.
> If the path doesn't exist yet, use `kv put` for the first write.

Verify it was stored:

```bash
bao kv get -address=http://192.168.1.210:8200 secret/homelab/ci
```

### 2. Check the CI policy

The CI AppRole uses a policy that grants read access to `secret/homelab/ci`. If you
added the credential at that path, no policy change is needed.

If you're adding a credential at a **new path**, update the policy:

```bash
bao policy read -address=http://192.168.1.210:8200 ci-policy
```

Add a new `path` block if required, then write the updated policy:

```bash
bao policy write -address=http://192.168.1.210:8200 ci-policy - <<EOF
path "secret/data/homelab/ci" {
  capabilities = ["read"]
}
path "secret/data/homelab/my-new-path" {
  capabilities = ["read"]
}
EOF
```

### 3. Read the credential in a CI workflow

In any Forgejo Actions workflow, fetch the new key after the AppRole login step:

```yaml
- name: Fetch secrets from OpenBao
  run: |
    TOKEN=$(curl -s -X POST $OPENBAO_ADDR/v1/auth/approle/login \
      -d "{\"role_id\":\"$OPENBAO_ROLE_ID\",\"secret_id\":\"$OPENBAO_SECRET_ID\"}" \
      | jq -r '.auth.client_token')
    MY_NEW_KEY=$(curl -s -H "X-Vault-Token: $TOKEN" \
      $OPENBAO_ADDR/v1/secret/data/homelab/ci \
      | jq -r '.data.data.MY_NEW_KEY')
    echo "MY_NEW_KEY=$MY_NEW_KEY" >> $GITHUB_ENV
```

> `OPENBAO_ADDR`, `OPENBAO_ROLE_ID`, and `OPENBAO_SECRET_ID` are Forgejo repository
> secrets set once — they do not change when adding new credentials.

### 4. Rotate a credential

To update an existing value without touching other keys:

```bash
bao kv patch -address=http://192.168.1.210:8200 secret/homelab/ci MY_NEW_KEY=new-value
```

No CI workflow changes needed — the new value is fetched at runtime on the next run.

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| CI returns `403 permission denied` | Key is at a path not covered by `ci-policy` — check Step 2 |
| CI returns `nil` for the value | Key name mismatch — verify with `bao kv get` |
| OpenBao returns `503` | Pod is sealed — check `bao status`; auto-unseal sidecar should recover within 30 s |
| `bao` CLI not found locally | Install: `brew install openbao` or use `vault` (API-compatible) |
