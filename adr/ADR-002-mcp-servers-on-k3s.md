# ADR-002: MCP Servers Run on K3s, Not on VM 102

## Status
Accepted

## Date
2026-04-03

## Context

MCP servers (forgejo-mcp, proxmox-mcp, mcp-atlassian) were originally deployed on VM 102 as ad-hoc Docker containers managed by ToolHive. This created a tight coupling between MCP server availability and VM 102's lifecycle:

- Any VM 102 restart killed all MCP connections and required manual reconfiguration
- There was no declarative record of how MCP servers were configured
- Adding a new MCP server required manual SSH and CLI commands with no audit trail

VM 102 also hosts Forgejo. Keeping all workloads on VM 102 creates a circular dependency risk: if Forgejo ran on K3s and K3s failed, the code needed to rebuild K3s would be inaccessible.

## Decision

**All MCP servers run as K3s workloads. VM 102 hosts Forgejo only.**

MCP servers are defined as Kubernetes manifests in the `infra` repo under `kubernetes/mcp-servers/` and registered with ArgoCD. This means:

- MCP server configuration is version-controlled and reviewed via PR
- K3s automatically restarts failed MCP pods without manual intervention
- MCP servers are completely decoupled from VM 102's lifecycle
- Adding a new MCP server follows the standard IaC workflow (branch → PR → merge → ArgoCD reconciles)

**VM 102 is intentionally isolated from K3s.** Forgejo must not move to K3s. If K3s fails, the code needed to rebuild it must remain accessible — Forgejo on VM 102 provides this guarantee.

## Consequences

- MCP server availability is no longer tied to VM 102 restarts
- All MCP configuration is auditable via git history
- New MCP servers must be added via PR to `infra` — no ad-hoc Docker commands
- VM 102 has a single, well-understood responsibility (Forgejo only), making it easier to reason about and recover from failures
- The FastMCP `/sse` vs `/mcp` path distinction must be accounted for when registering new Python-based MCP servers in Claude Code
