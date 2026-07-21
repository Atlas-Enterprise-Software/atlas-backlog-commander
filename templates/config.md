# <alias> — Azure DevOps config

> Template: copy to `<alias>/config.md` (one folder per project at the workspace root).
> Fill in the values marked `<TODO>` the first time the flow is used.
> The rest of the workspace reads this file as the source of truth.

## ADO identity

- **alias**: `<alias>`
- **mcp_server**: `<MCP server name>`
- **organization**: `<org>`
- **project**: `<project or TODO — pick with core_list_projects during bootstrap>`
- **default_area_path**: `<TODO — after first snapshot>`

## Process

> Detect during bootstrap with `wit_list_backlogs` + `wit_get_work_item_type` (step 0 of `workflows/bootstrap.md`).

- **process_template**: `<TODO>` (base: Scrum / Agile / Basic / CMMI / custom)
- **work_item_types**:
  - epic: `<TODO>`
  - feature: `<TODO — empty if the process does not have this level>`
  - requirement: `<TODO — Scrum: Product Backlog Item · Agile: User Story · Basic: Issue · CMMI: Requirement>`
  - bug: `<TODO>`
  - task: `<TODO>`
- **states** (key names match the workflow placeholders `<closed_states>` / `<removed_states>` / `<inactive_parent_states>`):
  - closed_states: `<TODO — e.g. Done, Closed>`
  - removed_states: `<TODO — e.g. Removed>`
  - inactive_parent_states: `<TODO — e.g. Done, Closed, Removed, Inactive>` (golden rule 5)
- **backlog_levels**: `<TODO — e.g. Epic → Feature → PBI/Bug>`

## Main area paths

`<TODO — fill in after first snapshot>`

## Iterations

- **Pattern**: `<TODO>`

## Tags convention

`<TODO — fill in after first snapshot>`

## WIQL base filter

- **wiql_base_filter** (reusable clause; workflows reference it by this key):

```sql
[System.TeamProject] = '<project>'
AND [System.State] NOT IN ('Removed')
```

## Local repositories

- **repos_root**: `<TODO — e.g. ~/Repos/<Org>>`
  - Rule: **reuse** `<repos_root>/<RepoName>` if already cloned; **clone** only if it does not exist.
- **clone_url_pattern**: `https://<org>@dev.azure.com/<org>/<project url-encoded>/_git/<RepoName>`

## Item creation defaults

- **Default iteration when creating**: backlog (no iteration assigned) unless the user specifies one.
- **Default Assigned To**: unassigned (the user decides).
- **bug_default_priority**: 2
- **bug_default_severity**: 3 - Medium
