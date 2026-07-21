# Atlas Backlog Commander — Global rules

**Atlas Backlog Commander**: Atlas tool for managing Azure DevOps backlogs with Claude, for **multiple organizations/projects**.
The project registry lives in **`projects.md`** (root). Active project resolution:

1. No project mentioned in the prompt → use the one marked `(default)` in `projects.md`.
2. "work with `<alias>`" → that project becomes the active one for the rest of the session.
3. "make `<alias>` the default" → move the `(default)` marker in `projects.md`.

`<project>` in this document = folder of the active project (e.g. `cb/`, `atlas/`).

## Sources of truth

- `projects.md` — project registry (alias, dir, org, ADO project, mcp_server, process, state).
- `<project>/config.md` — ADO org, project, **`mcp_server`**, **Process** section (work item types + states of the process template), area paths, conventions, **`repos_root`** (path to repos).
- `<project>/repos.md` — inventory of repos and tech stack.
- `<project>/backlog-snapshot.md` — compact index of the backlog (refreshable). **Search before creating any work item** to avoid duplicates (see golden rule 2). Carries YAML frontmatter with `last_updated` / `last_full_refresh`.
- `<project>/epics-tree.md` — hierarchical view Epic→Feature→top-level PBI (or whatever levels the process has).
- `workflows/*.md` — **shared** actionable templates (parameterized per project): create items, refresh snapshot, project onboarding.

## Golden rules

0. **First use (bootstrap)**: if `<project>/.bootstrap-done` for the active project does **not** exist, before creating/querying work items run `workflows/bootstrap.md` (discovers the process, scans **all** the repos of the DevOps project + the **entire** backlog), after user confirmation.
1. **Titles and descriptions always in English** (aligned with `atlas-webapi-backlog-generator`).
2. **Before creating a work item**, check `<project>/backlog-snapshot.md` for duplicates by title/area — **with Grep, not a full Read**: search the snapshot for the title's keywords (and synonyms) and the target area, then read only the matching rows/sections. Same with `<project>/epics-tree.md` when choosing a parent. A full Read is acceptable only for small snapshots (< 300 lines).
3. **Always show a tree preview** before creating in ADO and wait for user confirmation.
4. **Never assume** org or project if the command needs it and it is not in `config.md`: ask for it.
5. **When choosing the parent** of a work item, ignore candidates in the `inactive_parent_states` from the project's config (typically: `Done/Closed/Removed/Inactive`); if no suitable active parent remains, **propose creating a Feature** (`workflows/create-feature.md`).
6. **Repos**: reuse what is already cloned under `repos_root`; clone only what is missing. To analyze a repo's structure, **direct tree reading** via `workflows/analyze-repo.md` (layers/entities/endpoints).
7. **Every backlog item (and Bug) is associated with a repository**, indicated in its description (`## Repository` section). If the user does not specify it, infer the best fit by metadata matching (name/area/tech stack) over the available repos, propose 2-3 candidates and confirm.
8. **Never mix projects**: every creation/edit preview indicates the target project (`[project: <alias> — <org>/<project>]`). On any ambiguity about which project an item goes to, ask before writing to ADO.

## Workflow execution (multi-project context)

Every workflow in `workflows/*.md` runs under the same context (stated once here, not repeated per file):

- Resolve the active project via `projects.md` (root) and read `<project>/config.md` before executing — including its **Process** section.
- Every MCP function is invoked on the server from that config: `mcp__<mcp_server>__<function>`.
- Every creation/edit preview states the target project: `[project: <alias> — <org>/<project>]` (golden rule 8).

## Process per project

Each ADO project may use a different process template (Scrum, Agile, Basic, CMMI or custom), which changes **work item types** (`Product Backlog Item` vs `User Story` vs `Issue`…), **states** (`Done` vs `Closed`…) and **hierarchy** (Basic has no Feature). The actual names live in the `## Process` section of `<project>/config.md` and the workflows use them as placeholders — never hardcode types/states of a specific process in the workflows. **Placeholder names match the config keys exactly**: `<epic>`, `<feature>`, `<requirement>`, `<bug>`, `<task>` (from `work_item_types`) and `<closed_states>`, `<removed_states>`, `<inactive_parent_states>` (from `states`). Fields specific to one template (e.g. `Microsoft.VSTS.Common.AcceptanceCriteria`, `Severity`, `ReproSteps`) exist only where the process defines them — the workflows say how to degrade when they don't.

## Tool selection for creating items

| Case | Tool |
|------|-------------|
| New .NET Web API (Clean Architecture) | skill `atlas-webapi-backlog-generator` |
| Individual CRUD endpoint on an existing API | skill `atlas-webapi-crud-operation` |
| New Blazor module | skill `atlas-blazor-backlog-new-module` |
| New Blazor page/route | skill `atlas-blazor-backlog-new-page` |
| New reusable Blazor component | skill `atlas-blazor-backlog-new-component` |
| Blazor grid/listing | skill `atlas-blazor-backlog-grid-page` |
| Blazor side panel editor | skill `atlas-blazor-backlog-side-panel-editor` |
| Bug with a visible error in production | skill `atlas-appinsights-failures` for diagnosis → then `workflows/create-bug.md` |
| Strategic Epic | `workflows/create-epic.md` |
| Business / cross-cutting Feature | `workflows/create-feature.md` |
| Generic backlog item (non-CRUD, non-Blazor) | `workflows/create-pbi.md` |
| Generic Bug | `workflows/create-bug.md` |
| Batch of items described in `<project>/backlog.txt` | `workflows/process-backlog-file.md` (one agent per item in parallel, consolidated preview + confirmation, final summary) |
| Onboarding of a new ADO org/project | `workflows/add-project.md` |

> Note: the `atlas-*` skills generate items with Scrum types (PBI). They apply as-is only in projects whose `requirement` is `Product Backlog Item`; in other processes, use the generic workflows or map the type when creating.

## MCP Azure DevOps — functions to use

Functions are invoked on the **MCP server of the active project**: prefix `mcp__<mcp_server>__*`, with `mcp_server` read from `<project>/config.md` (e.g.: cb → `mcp__azure-devops__*`, atlas → `mcp__azure-devops-atlas__*`). If the org has no registered MCP server → `workflows/add-project.md` step 2.

- Backlog reading: `wit_query_by_wiql`, `wit_get_work_items_batch_by_ids`, `wit_list_backlog_work_items`.
- Creation: `wit_create_work_item` + `wit_add_child_work_items` or `wit_work_items_link` (relation `System.LinkTypes.Hierarchy-Reverse` for child→parent).
- Process discovery: `wit_list_backlogs`, `wit_get_work_item_type` (bootstrap step 0).
- Repos: `repo_list_repos_by_project`, `repo_list_directory`, `repo_get_file_content`, `search_code`. For **structure analysis** prefer cloning/reusing locally + direct tree reading (`workflows/analyze-repo.md`); the MCP `repo_*` functions remain as one-off reads/fallback.
- PR derived from a backlog item: skill `atlas-azure-devops-pr`.

> The workflows are also exposed as slash commands (`.claude/commands/`): `/create-pbi`, `/create-bug`, `/refresh-snapshot`, etc. — thin wrappers over `workflows/*.md`.

## Token efficiency

- `CLAUDE.md` is auto-loaded (warm cache). `projects.md` is small: read it when starting the first backlog operation.
- `<project>/config.md` is read **on activating** a project (once per session, unless it changes).
- `<project>/backlog-snapshot.md` and `<project>/epics-tree.md` are **searched (Grep), not fully read** — golden rule 2. Full reads only for small files.
- WIQL is used for specific deltas (a single item, comments, revisions), never to dump the entire backlog.
- Snapshot refresh: only when the user asks for it or when > 1 week has passed without updating (per the frontmatter's `last_updated`). By default **incremental**: WIQL delta with `ChangedDate >= last_updated` and merge onto the file. Full only on request, if there is no frontmatter or if `last_full_refresh` > 30 days (`workflows/refresh-snapshot.md`).
