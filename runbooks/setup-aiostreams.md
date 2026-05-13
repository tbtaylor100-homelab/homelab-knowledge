# Runbook: Set Up AIOStreams After Deployment

This runbook covers post-deploy UI configuration of AIOStreams v2.29.5. It guides an operator through verifying Real-Debrid connectivity, installing the Torrentio addon, and activating the pre-configured regex exclusion filter. AIOStreams must already be deployed and reachable at `http://192.168.1.205:3000` before following this runbook.

> **This runbook begins where `provision-aiostreams-secrets.md` ends.**
> Complete secrets provisioning (Phase 1) and Kubernetes deployment (Phase 2) before continuing here.

## Prerequisites

- AIOStreams pod is running and reachable:
  ```bash
  curl -s http://192.168.1.205:3000/api/v1/status
  ```
  Expected: HTTP 200 response with version field showing `v2.29.5`

- ExternalSecret has synced:
  ```bash
  kubectl get externalsecret -n aiostreams aiostreams-secret
  ```
  Expected: `READY: True`

- Stremio is installed on a LAN-connected device
- AIOStreams UI is accessible from your browser at: `http://192.168.1.205:3000`
- Secrets provisioning is complete per [provision-aiostreams-secrets.md](provision-aiostreams-secrets.md)

## Steps

### 1. Verify Real-Debrid is connected

Navigate to: **Services** menu → Real-Debrid row → click the **cogwheel icon** (⚙️)

The API Key field should be pre-populated from the `FORCED_SERVICE_CREDENTIALS` environment variable synced via ExternalSecret. Look for a **"Connected"** status indicator (green checkmark or "Connected" text).

If showing "Disconnected", see the Troubleshooting table below before continuing.

### 2. Install the Torrentio addon

Navigate to: **Addons** → **Marketplace** tab → search for "Torrentio"

If Torrentio does not appear in search results, use the custom manifest URL field and paste:

```
https://torrentio.strem.fun/manifest.json
```

Click **Install**. Verify the addon appears in the **Installed** tab after installation.

### 3. Activate the regex exclusion filter

Navigate to: **Filters** → **Regex Patterns** section

> **Note:** `WHITELISTED_REGEX_PATTERNS` is pre-configured via ConfigMap with the pattern
> `["/(WEB-DL|AMZN|DSNP|YTS|RARBG|EZTV)/i"]`. You do **not** need to re-enter the regex.

Enable the **"Use Regex Filters"** toggle (or equivalent UI control). Once activated, Stremio search results will exclude streams matching the pre-configured tags.

If the Regex Patterns section is not visible, verify the `REGEX_FILTER_ACCESS=all` environment variable is set (see Troubleshooting).

### 4. Add AIOStreams as a Stremio addon source

In Stremio on your LAN device:

1. Go to: **Settings** → **Addons** → **Add addon source** (or "Add via URL")
2. Enter the AIOStreams manifest URL:
   ```
   http://192.168.1.205:3000/manifest.json
   ```
3. Stremio will list available addons from AIOStreams, including the configured Torrentio instance.

### 5. Verify version (optional)

Confirm the running version matches these runbook steps:

```bash
kubectl exec -n aiostreams deployment/aiostreams -- curl -s http://localhost:3000/api/v1/status | jq '.version'
```

Expected: `"v2.29.5"`. If a different version is returned, menu names and field locations may differ from this runbook.

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| Real-Debrid shows "Disconnected" | ESO sync failure — check `kubectl get externalsecret -n aiostreams aiostreams-secret` for READY: True; restart pod with `kubectl rollout restart deployment aiostreams -n aiostreams` to force env var re-read |
| Regex Patterns section not visible in Filters | `ADDON_PASSWORD` may be required, or `REGEX_FILTER_ACCESS` env var is missing — verify with `kubectl exec -n aiostreams deployment/aiostreams -- env \| grep REGEX_FILTER_ACCESS` |
| Torrentio not found in Marketplace | Use manifest URL directly: `https://torrentio.strem.fun/manifest.json` in the custom URL field |
| Pod shows "Disconnected" after ESO sync confirms SecretSynced | Restart pod to force env var re-read: `kubectl rollout restart deployment aiostreams -n aiostreams`; wait 30s and refresh UI |
| UI menu names don't match this runbook | Version has changed from v2.29.5 — run Step 5 to verify version; refer to current AIOStreams docs at https://guides.viren070.me/stremio/addons/aiostreams/ |
| Stremio shows "no results" after addon add | Verify Torrentio manifest is reachable: `curl https://torrentio.strem.fun/manifest.json`; also ensure Torrentio is in the Installed tab in AIOStreams |
