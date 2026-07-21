# Atlas Backlog Commander — Projects

> Template: copy to the **workspace root** as `projects.md` and replace the example row with your real projects (guided onboarding: `workflows/add-project.md`).

Registry of the workspace's Azure DevOps projects. Each row points to a folder with its `config.md` (operational source of truth: ADO identity, process, repos, defaults).

| Alias | Dir | ADO Org | ADO Project | MCP server | Process | State |
|-------|-----|---------|-------------|------------|---------|--------|
| `<alias>` (default) | `<alias>/` | `<org>` | `<project or TODO: choose in bootstrap>` | `<mcp_server>` | `<TODO: detect in bootstrap>` | pending-bootstrap |

> The `Process` column is informative (base: Scrum / Agile / Basic / CMMI / custom). The operational detail (work item types and states) lives in the **Process** section of each project's `config.md`.

## Active project resolution

1. If the prompt **does not mention** a project → use the one marked `(default)`.
2. "work with `<alias>`" → that project becomes the **active one for the rest of the session**.
3. "make `<alias>` the default" → move the `(default)` marker in this table.

MCP functions are invoked on the active project's server: prefix `mcp__<mcp_server>__*` (MCP server column / `mcp_server` field of the config).

Onboarding a new project → `workflows/add-project.md`.
