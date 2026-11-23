# Workflow — Process backlog.txt (one agent per item, in parallel)

> Multi-project context: see `CLAUDE.md → Workflow execution`. The file to process is **`<project_dir>/backlog.txt`** (each project has its own). Pass each agent the project alias, the `mcp_server` and the type/state mapping from the **Process** section of the config.

Procedure Claude executes when the user says "process the backlog.txt", "generate the backlog from the file", "create the PBIs from backlog.txt" or similar.

General idea: each item described in `<project_dir>/backlog.txt` is analyzed and created by an **independent agent** (parallel processes). The flow has two phases separated by the user's confirmation (golden rule 3): parallel analysis → **consolidated preview + confirmation** → parallel creation → **final summary** when they all finish.

## Steps

### 1. File parsing

- Read `<project_dir>/backlog.txt`.
- Each bullet (`- `) or non-empty paragraph = **1** backlog item.
- **Ignore** already processed lines: those starting with `[OK #id]`.
- If the file is empty or everything is processed → inform and stop.

### 2. Pre-check: snapshot freshness

If `last_updated` (frontmatter of `<project_dir>/backlog-snapshot.md`) is more than 1 week old, first run `workflows/refresh-snapshot.md` in **incremental** mode, so dedup and parent selection run on reliable data.

### 3. Phase 1 — Analysis (parallel, NO writes to ADO)

Launch **one `general-purpose` agent per item**, all **in a single message** (multiple calls to the Agent tool) so they run in parallel. Each agent's prompt (include the item's literal text + project alias, `mcp_server` and the config's type/state mapping):

- Read `<project_dir>/config.md` and `<project_dir>/repos.md`. **Grep** (do not fully Read) `<project_dir>/backlog-snapshot.md` for duplicates by title/keywords and `<project_dir>/epics-tree.md` for parent candidates (golden rule 2).
- Classify the item using the "Tool selection" table in `CLAUDE.md`. If an `atlas-*` skill fits (Blazor page/grid/panel, Web API CRUD, etc.), use it **only to structure the draft** (titles, item hierarchy, AC per its templates). Inside a subagent there is no user interaction, so the skill **must not create anything** in ADO.
- Infer the **repository** (best candidate + 2-3 alternatives) by name/area/tech stack matching (golden rule 7).
- Infer the **active parent** — mandatory for every item (golden rule 5: discard `<inactive_parent_states>` states from config); if there is no valid parent, propose creating one (`workflows/create-feature.md`, or `create-epic.md` in processes without `<feature>`) as part of the proposal.
- Write the draft following the templates in `workflows/create-pbi.md` (steps 3 and 4): description with `## Repository` / `## Goal` / `## Technical notes` / `## Out of scope` and Acceptance Criteria.
- Return a **structured proposal** (markdown): item index, title/description/AC **in English**, work item type (actual name from the project's process), area path, proposed parent (ID + title + state), chosen repo + alternatives, detected duplicates (IDs) and open questions.
- **FORBIDDEN** to call `wit_create_work_item`, `wit_add_child_work_items`, `wit_update_work_item` or any ADO write in this phase.

### 4. Consolidated preview + confirmation

With all the agents' proposals, **work out the dependency chain** (golden rule 9): order the items in implementation order (by default, the order of `backlog.txt`); item *n* is the predecessor of item *n+1* when they share parent/area, items that can run in parallel share a predecessor instead, and the first item of each chain links to the active sibling it depends on, if any. The agents can't see each other's proposals, so this is decided here, not in Phase 1.

Then show **a single tree** (format from step 5 of `create-pbi.md`, one block per item), headed by the target project:

```
[project: <alias> — <org>/<project>]
[1] <requirement> (new) → Feature #{parentId} {parentTitle}
    └── Title: {title}
        Repo: {RepoName}
        Area: {area}
        AC: {n} bullets | Tags: {tags}
        Predecessor: [1] or #{id} {title} | Successor: [3] or #{id}   ← or "none"
        ⚠ Possible duplicate: #{id} {title}   ← only if applicable
[2] ...
```

Flag duplicates and open questions. An item without a parent is **blocking**: it is not created until the user picks a parent or approves creating one (created first, before the item). **Wait for the user's confirmation**: they may approve everything, exclude items or adjust the repo/parent of any of them. Create nothing without an OK.

### 5. Phase 2 — Creation (parallel)

Launch **one agent per approved item**, all in a single message. Each agent receives its approved draft (already final, without re-running interactive skills) and:

- Creates with `wit_create_work_item` on the project's MCP server (fields: `System.Title`, `System.AreaPath`, `System.Tags`, `System.Description`, `Microsoft.VSTS.Common.AcceptanceCriteria` — the latter only if the type has the field; otherwise the criteria go inside the description, see the guard in `create-pbi.md` step 4), using the process's actual `workItemType`.
- Links to the parent with `wit_work_items_link` (relation `System.LinkTypes.Hierarchy-Reverse`).
- Links its confirmed predecessor/successor **when it is an existing `#id`** (outside the batch), per `CLAUDE.md → Dependency links`.
- Returns the **ID + URL** of each created work item, or the **error** if something fails (do not retry more than once).

Dependency links between items of the batch need the IDs of both ends, so the creation agents only link the existing `#id` ends (above). Once **all** agents finish, link the in-batch chain yourself from the returned IDs — each item to its in-batch predecessor; that one link also gives the predecessor its successor, so the `[n]` successors in the preview need no separate link. If an item failed, skip its links and report the broken chain in the summary.

### 6. Final summary

When **all** the creation agents finish, show a consolidated table:

| # | Item (summary) | Work item | Parent | Predecessor | Repo | URL |
|---|----------------|-----------|--------|-------------|------|-----|

Include a failures section if there were any (item, cause, retry suggestion).

### 7. Post-create

- In `<project_dir>/backlog.txt`, mark each processed line with the prefix `[OK #id] ` (the ID of the main work item created). Failed lines are left unmarked.
- Add the new rows to `<project_dir>/backlog-snapshot.md` (corresponding section) and the branches to `<project_dir>/epics-tree.md`, and **bump the snapshot frontmatter's `last_updated` to now (ISO 8601)** — same as step 7 of `create-pbi.md`.
