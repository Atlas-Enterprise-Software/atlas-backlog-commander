# Workflow — Bootstrap (first time per project)

> Multi-project context: see `CLAUDE.md → Workflow execution`. `<project_dir>` / `<alias>` = dir and alias of the active project.

**One-time** initialization per project: scans **all repos** of the DevOps project and **the entire
backlog**, so that deduplication, repo→PBI matching and parent lookup work
with complete data from the very first use.

## When it runs

- **Automatically the first time** a work item of the active project is about to be created/queried, if the
  `<project_dir>/.bootstrap-done` marker does **not** exist.
- Manually if the user asks to "reinitialize" or do a "full rescan" (regenerates both sources).

Warn the user that it is a full scan (it may take a while) and **confirm** before launching it.

## Steps

### 0. Complete the project's identity and process

1. If `project` is set to `<TODO>` in `<project_dir>/config.md` → list with `core_list_projects` and ask the user to choose; persist it in config and in `projects.md`.
2. **Discover the process** (actual work item types and states, including custom processes):
   - `wit_list_backlogs` → backlog levels and types per level (epic / feature / requirement).
   - `wit_get_work_item_type` for each relevant type → valid states.
   - Fill in the `## Process` section of `<project_dir>/config.md` (`work_item_types`, `states`, `backlog_levels`) and the `Process` column of `projects.md`.

### 1. Scan all repos of the project

1. `repo_list_repos_by_project` (org/project from `<project_dir>/config.md`) → complete list of repos.
2. For each repo, infer metadata:
   - `tech_stack`: detect root files (`*.sln`/`*.csproj`, `package.json`, `host.json`) — via the local dir if cloned, or `repo_list_directory` otherwise.
   - `area_path`: from recent PRs (`repo_list_pull_requests_by_repo_or_project`) and their linked work items.
   - For cloned repos, the **structural reading** (layers/entities/endpoints) is done on-demand with `workflows/analyze-repo.md` when generating backlog for that repo — not during bootstrap.
3. Rewrite **`<project_dir>/repos.md`** with the complete list of the project's repos and the inferred `tech_stack`/area, preserving manual notes.

### 2. Scan the entire backlog

- Run the **`workflows/refresh-snapshot.md`** workflow in full (full WIQL queries) → regenerates `<project_dir>/backlog-snapshot.md` and `<project_dir>/epics-tree.md`, and copies a historical snapshot to `<project_dir>/history/`.

### 3. Mark as initialized

- Write `<project_dir>/.bootstrap-done` with the date and a summary (number of repos scanned, snapshot totals).
- Update `projects.md`: the project's State column → `bootstrapped`.
- From here on, the workflows use the already populated sources; no re-bootstrapping happens unless explicitly requested.

## Notes

- Subsequent backlog refreshes → `workflows/refresh-snapshot.md` (no need to re-scan repos every time).
- The repo inventory is refreshed when new repos are added to the project or the user asks for it.
