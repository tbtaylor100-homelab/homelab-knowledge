# Runbook: Provision AIOStreams Secrets in OpenBao

Use this runbook when setting up AIOStreams for the first time. It extends the
`eso-policy` to cover the AIOStreams secret path and provisions the two required
credentials at `secret/aiostreams/production` so that the ExternalSecret Operator
can sync them into Kubernetes.

> **This runbook is for initial provisioning only.**
> If AIOStreams is already running and you need to update the Real-Debrid API key
> without touching `SECRET_KEY`, use `bao kv patch` instead of `bao kv put` to
> avoid overwriting the immutable key. See Step 3 for details.

## Prerequisites

- OpenBao is unsealed and reachable — verify with:
  ```bash
  bao status -address=http://192.168.1.210:8200
  ```
  Output must show `Sealed: false`
- You have a root token or a token with `sys/policy/write` and `secret/data/*` write access
- `eso-policy.hcl` is committed to the infra repo at `kubernetes/external-secrets/eso-policy.hcl`
  (completed in Phase 1, Plan 01 — run that plan before this runbook)
- Your Real-Debrid API key is in hand — retrieve it from: https://real-debrid.com/apitoken

## Steps

### 1. Verify current policy state

Before applying, confirm what the live policy currently contains:

```bash
bao policy show -address=http://192.168.1.210:8200 eso-policy
```

Expected output shows only `secret/data/homelab/ci`. After Step 2, it will also show
`secret/data/aiostreams/*`.

### 2. Apply the extended eso-policy

From the root of the `infra` repo (`C:\repos\infra`):

```bash
bao policy write -address=http://192.168.1.210:8200 eso-policy kubernetes/external-secrets/eso-policy.hcl
```

Verify both paths are present in the applied policy:

```bash
bao policy show -address=http://192.168.1.210:8200 eso-policy
```

Expected: output includes both `secret/data/homelab/ci` and `secret/data/aiostreams/*` with
`capabilities = ["read"]`. If only one path appears, the HCL file is missing the other — do
not proceed until both are confirmed.

### 3. Generate SECRET_KEY

```bash
SECRET_KEY=$(openssl rand -hex 32)
echo "SECRET_KEY=$SECRET_KEY"
```

Record the output before proceeding. This value cannot be recovered from OpenBao in plaintext
after the initial write.

> **WARNING: SECRET_KEY is immutable.**
> Once AIOStreams starts and a user saves their configuration, `SECRET_KEY` CANNOT be changed
> without invalidating all saved user configurations. Generate it once here and never rotate it.
> If you ever need to update `FORCED_SERVICE_CREDENTIALS` later, use `bao kv patch` — never
> run `bao kv put` on `secret/aiostreams/production` again after initial provisioning.

### 4. Write both secrets atomically

Replace `<YOUR_RD_API_KEY>` with the key from https://real-debrid.com/apitoken.

Both fields are written in a single command to avoid partial state:

```bash
bao kv put -address=http://192.168.1.210:8200 secret/aiostreams/production \
  SECRET_KEY="$SECRET_KEY" \
  FORCED_SERVICE_CREDENTIALS="realdebrid.apiKey=<YOUR_RD_API_KEY>"
```

> Use `kv put` (not `kv patch`) for this initial write — the path `secret/aiostreams/production`
> does not yet exist. `kv put` creates it. For any subsequent updates to `FORCED_SERVICE_CREDENTIALS`,
> use `kv patch` to avoid overwriting `SECRET_KEY`.

### 5. Verify the stored secret

```bash
bao kv get -address=http://192.168.1.210:8200 secret/aiostreams/production
```

Expected output shows two fields:
- `SECRET_KEY` — 64-character hex string
- `FORCED_SERVICE_CREDENTIALS` — value is `realdebrid.apiKey=<your-key>` (not just the key alone)

Phase 1 is complete. Phase 2 (Kubernetes manifests) can now begin — the ExternalSecret will
sync these fields without a permission error.

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| `503 Service Unavailable` from bao | OpenBao pod is sealed — run `bao operator unseal -address=http://192.168.1.210:8200` |
| `403 permission denied` on `bao policy write` | Token lacks `sys/policy/write` — use root token |
| `bao policy show` shows only one path after Step 2 | HCL file is missing the other path — verify `eso-policy.hcl` contains both `homelab/ci` and `aiostreams/*` |
| `bao kv get` returns empty or missing fields | Path typo — verify `secret/aiostreams/production` (no `/data/` in CLI path) |
| ESO sync shows `permission denied` in Phase 2 | Policy not applied (Step 2 was skipped), or policy was applied from an incomplete HCL file missing `aiostreams/*` |
| `Error making API request: dial tcp 127.0.0.1:8200` | Missing `-address` flag — bao CLI defaults to localhost; all commands need `-address=http://192.168.1.210:8200` |
