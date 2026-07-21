# Workflow — Add Project (onboard a new ADO org/project)

Onboards a new project into the workspace: registration in `projects.md`, its own folder with `config.md` and (if needed) an MCP server for its organization.

## Steps

### 1. Gather inputs

| Field | Required | Notes |
|-------|-----------|-------|
| Alias | yes | short, kebab/lowercase (e.g. `cb`, `atlas`); it will be the folder name |
| ADO organization | yes | e.g. `AtlasEnterpriseSoftware` |
| ADO project | optional | if unknown, it is picked during bootstrap with `core_list_projects` |
| repos_root | optional | local folder for clones; suggest `~/Repos/<Org>` |

Check that the alias does not already exist in `projects.md`.

### 2. Org MCP server

1. Check whether there is already an MCP server pointing to that org (tools with prefix `mcp__<server>__*` available and correct org).
2. If it does **not** exist → ask the user to register it at user level (any server name works; `<alias>-ado` is just a suggestion) and restart the session:
   ```bash
   claude mcp add <server-name> --scope user -- npx -y @azure-devops/mcp <org>
   ```
   Whatever name they choose is the value of `mcp_server` in the config and in `projects.md` (existing examples: `azure-devops`, `azure-devops-atlas`).

### 3. Create `<alias>/config.md` from the template

Copy **`templates/config.md`** to `<alias>/config.md` (single source of truth for the config structure — do not maintain a copy of it here) and fill in what is already known: `alias`, `mcp_server`, `organization`, and `project`/`repos_root` if provided. Leave the rest as `<TODO>` for bootstrap.

### 4. Register in `projects.md`

Add a row with Status `pending-bootstrap`. Do not touch the `(default)` mark unless the user asks for it.

### 5. Next step

Suggest running `workflows/bootstrap.md` for the new project (fills in project/process/areas, scans repos and generates snapshot + epics-tree).
