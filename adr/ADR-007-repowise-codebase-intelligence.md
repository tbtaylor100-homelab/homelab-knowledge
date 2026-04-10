# ADR-007: RepoWise for Codebase Intelligence and Planning Context

## Status
Accepted

## Date
2026-04-10

## Context

Planning and implementation sessions with Claude Code require the agent to build
context about the homelab codebase from scratch at the start of every conversation.
For the `infra`, `homelab-plans`, and `homelab-knowledge` repos, this means the agent
reads many files — playbooks, manifests, ADRs, workflows — before it can reason about
a task. This upfront exploration consumes a significant portion of the context window
and incurs unnecessary token costs on every session.

The problem compounds with planning tasks: the agent must trace existing patterns
(e.g. "how are MCP server secrets created?"), understand repo structure, and locate
relevant files before it can write a single line. All of that exploration is repeated
every session, even though the codebase changes infrequently between sessions.

## Decision

**Deploy RepoWise as a K8s workload to provide a persistent, pre-indexed codebase
intelligence layer for all homelab repos.**

RepoWise (PyPI: `repowise==0.2.1`) indexes repositories once, builds a structured
knowledge base from the code and git history, and exposes that knowledge via an MCP
server. Claude Code consumes the MCP tools (`get_overview`, `get_context`,
`search_codebase`, `get_why`, etc.) instead of reading raw files, which:

- Eliminates the per-session repo exploration cost
- Reduces input token consumption for planning and implementation sessions
- Provides architectural decision search (`get_why`) so the agent can locate ADRs
  without reading the full `homelab-knowledge` repo
- Keeps context focused on the task rather than codebase archaeology

RepoWise runs in the `mcp-servers` namespace alongside the existing Forgejo, Proxmox,
and Atlassian MCP servers. The deployment follows the same pattern: Kubernetes
Deployment + LoadBalancer Service, managed by ArgoCD. A PVC preserves the index
across pod restarts. A background loop runs `repowise update` hourly so the index
stays current without manual intervention.

## Consequences

- Claude Code sessions for homelab work should invoke RepoWise MCP tools for initial
  context before exploring raw files, reducing token usage per session
- The initial `repowise init` takes ~25 minutes per repo; the pod must be given
  sufficient readiness probe grace (30 min) to avoid crash-looping during first deploy
- A custom Docker image (`192.168.1.50:3000/root/repowise:0.2.1`) must be built and
  maintained; version bumps require a new image build via Forgejo Actions CI
- The repos to index are configured via a K8s secret (`REPOWISE_REPOS`); adding a new
  repo requires updating the secret and restarting the pod to trigger cloning and init
- RepoWise uses `mcp-proxy` to bridge its stdio MCP transport to HTTP, consistent with
  the proxmox-mcp pattern established in ADR-002
