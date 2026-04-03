# ADR-001: Documentation Location Policy

## Status
Accepted

## Date
2026-04-03

## Context

As the homelab grows across multiple repos (`infra`, app repos, `homelab-knowledge`, `homelab-plans`), documentation will accumulate. Without a clear policy, docs end up scattered — operational guides buried in ADRs, architectural decisions buried in repo READMEs, or knowledge duplicated across repos. This makes it hard for future sessions (human or agent) to know where to look or where to write.

## Decision

Documentation is split by scope:

**Repo-scoped documentation → lives in the repo it documents**

Operational knowledge that has no meaning outside a single repo belongs in that repo's `docs/` directory. Examples:
- How to add a new MCP server to `infra`
- What environment variables an app's Helm chart accepts
- How to run a specific Ansible playbook

This keeps repos self-contained. Someone cloning `infra` has everything they need to operate it without consulting another repo.

**Cross-cutting architectural decisions → live in `homelab-knowledge` as ADRs**

Decisions that establish patterns affecting multiple repos, services, or future work belong in `homelab-knowledge`. Examples:
- Why MCP servers run on K3s rather than VM 102
- Why documentation follows this split policy
- Why a particular tool was chosen over alternatives

`homelab-knowledge` is the single authoritative source for *why* the homelab is shaped the way it is.

**The test for which category a document belongs to:**
> "Would someone working only in repo X need this document to do their job?"
> - Yes → it belongs in repo X's `docs/`
> - No, it affects how multiple things are built → it belongs in `homelab-knowledge` as an ADR

## Consequences

- Each repo is operationally self-contained for its own concerns
- `homelab-knowledge` never contains step-by-step operational guides for a single repo
- Repo `docs/` directories never contain architectural rationale that spans multiple services
- Future Claude Code sessions should check `homelab-knowledge` for architectural context and the relevant repo's `docs/` for operational how-tos
