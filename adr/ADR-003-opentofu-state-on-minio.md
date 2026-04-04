# ADR-003: OpenTofu State Stored in MinIO on Alexandria NAS

## Status
Accepted

## Date
2026-04-04

## Context

During Epic 1 bootstrapping, OpenTofu state for all stacks was stored as a local
`.tfstate` file on the operator's Mac (gitignored). This was a deliberate
constraint: state can't live in MinIO before MinIO exists, and MinIO can't exist
before K3s exists.

Local state has real operational risks:

- **Single machine dependency** — if the operator's Mac is unavailable, no one
  can run `tofu plan` or `tofu apply`
- **No concurrent-run protection** — without a state lock backend, two
  simultaneous applies could corrupt state
- **No audit trail** — there is no history of who ran what and when

With K3s running and MinIO deployed on Alexandria NAS (192.168.1.132), the
bootstrap constraint is resolved.

## Decision

**All OpenTofu state is stored in MinIO on Alexandria NAS under the
`opentofu-state` bucket.**

Each stack has its own key under a common bucket:

| Stack | State key |
|---|---|
| k3s | `k3s/terraform.tfstate` |
| uptime-kuma | `uptime-kuma/terraform.tfstate` |

The S3 backend is configured with `force_path_style = true` and AWS validation
checks disabled (`skip_credentials_validation`, `skip_metadata_api_check`,
`skip_region_validation`) because MinIO speaks the S3 API but does not implement
AWS-specific IAM/STS endpoints.

Credentials (access key and secret key) are **not committed to the repo**. They
are passed via environment variables (`AWS_ACCESS_KEY_ID`,
`AWS_SECRET_ACCESS_KEY`) sourced from a gitignored local file before running
tofu. A dedicated `opentofu` access key is scoped to the `opentofu-state` bucket
only (principle of least privilege).

MinIO is also used for Velero backups (MAH-62) under a separate `velero` bucket
with separate credentials. Buckets and keys are fully isolated between use cases.

## Consequences

- OpenTofu state is durable and accessible from any operator machine on the
  homelab network
- The `opentofu-state` bucket must exist in MinIO before `tofu init` can succeed
- `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` must be exported in the
  operator's shell before running any tofu command
- MinIO on Alexandria becomes a hard dependency for all tofu operations —
  acceptable because the NAS runs 24/7
- Future operators must be given the `opentofu` credentials out-of-band (or via
  OpenBao once that epic is complete)
- State locking is not yet configured (MinIO does not natively provide DynamoDB
  locking) — concurrent applies remain a risk, mitigated by convention (one
  operator at a time)
