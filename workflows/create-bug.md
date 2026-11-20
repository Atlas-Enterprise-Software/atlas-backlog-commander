# Workflow — Create Bug

> Multi-project context: see `CLAUDE.md → Workflow execution`. Relevant config: the **Process** section (`bug` type and states).
>
> **Process guard**: if the active project has no `bug` type (e.g. Basic) → create it as the `<requirement>` type (`Issue`…) with the `bug` tag, **omit** the `Microsoft.VSTS.Common.Severity` / `Microsoft.VSTS.Common.Priority` / `Microsoft.VSTS.TCM.ReproSteps` fields (they don't exist in that process — Severity/Env go in the description's `## Environment` / `## Summary`), and state the substitution in the preview. Do not invent the type.

## Steps

### 1. Pre-checks
1. Check the "Active Bugs" section of `<project_dir>/backlog-snapshot.md` for already reported ones (same symptom/area) — **Grep by symptom keywords and area; do not Read the whole file** (golden rule 2).
2. If the bug comes from a visible production error:
   - Suggest the user invoke the `atlas-appinsights-failures` skill first for diagnosis → reuse the analysis as `Technical notes`.
3. **Identify the repository**: the bug must be associated with a repository.
   - If the user specifies it → use it (validate against `<project_dir>/repos.md` / `repo_list_repos_by_project`).
   - If they do **not** specify it → infer by name / `area_path` / `tech_stack` matching (`<project_dir>/repos.md` inventory + ADO list + dirs under `repos_root`) against the bug's symptom/area; propose 2-3 candidates and confirm.
   - Reflect it in the description (`## Repository` section) as a link to the repo: the `{repo web url}` in the template is the `clone_url_pattern` from config without the `<org>@` user prefix. If config has no `clone_url_pattern`, write just the repo name as plain text (no link).
4. Parent (optional): if it will be linked to a Feature/backlog item, **discard** candidates with State ∈ `<inactive_parent_states>` from config (verify live with `wit_get_work_item` if in doubt). If no suitable active parent remains and the bug needs framing → propose creating a Feature (`workflows/create-feature.md`); otherwise, leave it without a parent (valid for bugs).

### 2. Gather inputs

| Field | Required | Default |
|-------|----------|---------|
| Title (English) | yes | — |
| Repository | yes | inferred by matching and confirmed (see Pre-checks) |
| Area Path | yes | area of the associated repo |
| Severity | yes | config default (`bug_default_severity`) |
| Priority | yes | config default (`bug_default_priority`) |
| Environment (DEV/PRE/PRO) | yes | — |
| Repro steps | yes | — |
| Expected vs Actual | yes | — |
| Affected version / build | recommended | — |
| Logs / App Insights operation_Id | recommended | — |
| Parent (Feature or backlog item) | optional | — |
| Tags | optional | add `bug-prod` if environment=PRO |

### 3. `System.Description` template

```markdown
## Repository
[{RepoName}]({repo web url})

## Summary
{one sentence: what fails, where, impact}

## Environment
- Stage: {DEV | PRE | PRO}
- Build / Version: {x.y.z}
- Operation Id / Trace Id: {if any}

## Repro steps
1. {step}
2. {step}
3. {step}

## Expected
{what should happen}

## Actual
{what happens}

## Technical notes
{stack trace, hypothesis, suspect file, App Insights findings}
```

### 4. `Microsoft.VSTS.TCM.ReproSteps`

Identical copy of the "Repro steps + Expected + Actual" section in this field (Scrum/Agile processes handle it separately; if the project's process lacks the field, omit it).

### 5. Mandatory preview

```
[project: <alias> — <org>/<project>]
BUG (new) [Sev: {sev} | Pri: {pri} | Env: {env}]
└── Title: {title}
    Repo: {RepoName}
    Area: {area}
    Parent: {parentId or "none"}
    Tags: {tags}
```
Wait for confirmation.

### 6. Create in ADO

`wit_create_work_item` with:
- `workItemType`: `bug` type from config (usually `Bug`; if empty → `<requirement>` + `bug` tag, see Process guard)
- `fields`:
  - `System.Title`, `System.AreaPath`, `System.Tags`
  - `System.Description`, `Microsoft.VSTS.TCM.ReproSteps`
  - `Microsoft.VSTS.Common.Severity`: severity
  - `Microsoft.VSTS.Common.Priority`: priority
  - (the three `Microsoft.VSTS.*` fields only where the process defines them — see Process guard)

Link to parent if it exists (`wit_work_items_link`, `System.LinkTypes.Hierarchy-Reverse`).

### 7. Post-create

- Report ID/URL.
- If Sev = `1 - Critical` → remind the user to notify the corresponding channel, regardless of environment (do not send automatically). If env = PRO, highlight the reminder as a production incident (`bug-prod` tag already applied in step 2).
- Update the row in `<project_dir>/backlog-snapshot.md` and **bump its frontmatter's `last_updated` to now (ISO 8601)**.
