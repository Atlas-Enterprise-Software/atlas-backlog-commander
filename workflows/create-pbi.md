# Workflow — Create backlog item (generic)

> Multi-project context: see `CLAUDE.md → Workflow execution`. `<requirement>` = the process's requirement type from config (`Product Backlog Item` in Scrum, `User Story` in Agile, `Issue` in Basic…).

For backlog items that are **NOT** .NET Web API CRUD or standard Blazor developments (those go through the `atlas-webapi-*` and `atlas-blazor-*` skills). Useful for: integration tasks, refactors, observability, tech debt, automations, etc.

## Steps

### 1. Pre-checks
1. Check `<project_dir>/backlog-snapshot.md` for duplicates by title or keywords — **Grep for the title's keywords/synonyms and the target area; do not Read the whole file** (golden rule 2). Read only the matching rows and their surrounding section.
2. **Identify the repository (mandatory)**: every item must be associated with a repository.
   - If the user specifies it → use it (validate that it exists in `<project_dir>/repos.md` / `repo_list_repos_by_project`).
   - If they do **not** specify it → infer the best fit: rank the available repos (`<project_dir>/repos.md` inventory + full list from `repo_list_repos_by_project`, plus dirs under `repos_root`) by name / `area_path` / `tech_stack` affinity with the item's goal and area. Propose **2-3 candidates** and **confirm** with the user before proceeding.
   - The chosen repo is reflected in the description (`## Repository` section) as a link to the repo: the `{repo web url}` in the template is the `clone_url_pattern` from config without the `<org>@` user prefix. If config has no `clone_url_pattern`, write just the repo name as plain text (no link). Optional: dig deeper with `workflows/analyze-repo.md` (structure reading) if the fit needs validating.
3. Identify the parent Feature (or whatever parent level the process has):
   - If the user mentions a specific module/repo → use `<project_dir>/repos.md` to map repo → area path → Features under that area in `<project_dir>/epics-tree.md` (**Grep the tree by area/keywords; do not Read it whole**).
   - **Discard** as a candidate any parent with State ∈ `<inactive_parent_states>` from config (only genuinely active parents are valid). The snapshot's State column enables the filter; if you doubt its freshness, verify live with `wit_get_work_item` before linking.
   - If ambiguous → ask the user and propose 2-3 candidates (active only).
   - **If after filtering no suitable active parent remains → propose creating one** (`workflows/create-feature.md`, or `create-epic.md` in processes without `<feature>`) and create it first. **The parent is mandatory** (golden rule 5): never create the item without one.
4. **Predecessor / successor** (golden rule 9): among the parent's active children (Grep `epics-tree.md`) and the other items of the same request, pick the item that must be done first and the one that depends on this one. Several items in one request → chain them in implementation order.

### 2. Gather inputs

| Field | Required | Default |
|-------|----------|---------|
| Title (English) | yes | — |
| Repository | **yes** | inferred by matching and confirmed (see Pre-checks) |
| Parent Feature ID | **yes** | — (golden rule 5) |
| Area Path | yes | parent Feature's area or derived from the repo |
| Goal (1-2 sentences) | yes | — |
| Acceptance criteria | yes | — |
| Technical notes | optional | — |
| Tags | optional | — |
| Effort estimate | optional | unestimated |

### 3. `System.Description` template

```markdown
## Repository
[{RepoName}]({repo web url})

## Goal
{what this item solves and why}

## Technical notes
{files to touch, dependencies, risks}

## Out of scope
- {item}
```

### 4. Acceptance criteria — `Microsoft.VSTS.Common.AcceptanceCriteria`

> **Process guard**: this field exists in Scrum/Agile/CMMI. If the process's `<requirement>` type lacks it (e.g. Basic `Issue`), put the same content in an `## Acceptance criteria` section at the end of `System.Description` instead.

```markdown
- [ ] {observable criterion}
- [ ] {observable criterion}
- [ ] Tests cover the happy path
- [ ] Tests cover at least one error/edge case
```

### 5. Mandatory preview

```
[project: <alias> — <org>/<project>]
<requirement> (new) → Feature #{parentId} {parentTitle}
└── Title: {title}
    Repo: {RepoName}
    Area: {area}
    AC: {n} bullets | Tags: {tags}
    Predecessor: #{id} {title} | Successor: #{id} {title}   ← or "none"
```
Wait for confirmation.

### 6. Create in ADO

`wit_create_work_item` with:
- `workItemType`: `requirement` type from config
- `fields`:
  - `System.Title`, `System.AreaPath`, `System.Tags`
  - `System.Description`: description
  - `Microsoft.VSTS.Common.AcceptanceCriteria`: criteria (only if the type has the field — see step 4 guard)

Link to the parent Feature with `wit_work_items_link` (`System.LinkTypes.Hierarchy-Reverse`).

Then link the confirmed predecessor/successor (`CLAUDE.md → Dependency links`). Several items → create them in predecessor order.

### 7. Post-create

- Report ID/URL.
- If the user wants to start development immediately → offer to create branch `pbi-<id>` (skill `atlas-azure-devops-pr` or `repo_create_branch`).
- Update the row in `<project_dir>/backlog-snapshot.md` and the tree in `<project_dir>/epics-tree.md`, and **bump the snapshot frontmatter's `last_updated` to now (ISO 8601)** — every merge must move the date, or incremental refresh will re-fetch (or miss) these items.
