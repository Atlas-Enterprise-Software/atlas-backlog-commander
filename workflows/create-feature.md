# Workflow — Create Feature

> Multi-project context: see `CLAUDE.md → Workflow execution`.
>
> **Process guard**: if the active project has no `feature` type (e.g. Basic process), warn the user and offer to create the parent level that does exist (`epic`) or link the item directly to the upper level. Do not invent the type.

For business or cross-cutting Features not covered by `atlas-*` skills (the atlas-webapi and atlas-blazor skills already create Features automatically when applicable).

## Steps

### 1. Pre-checks
1. Check the Features section of `<project_dir>/backlog-snapshot.md` for duplicates by title/area — **Grep by keywords/area; do not Read the whole file** (golden rule 2).
2. Identify the parent Epic: **Grep `<project_dir>/epics-tree.md`** by area/keywords and propose the best-fitting Epic. **Discard** Epics with State ∈ `<inactive_parent_states>` from config (only active Epics are valid); if you doubt the snapshot's freshness, verify live with `wit_get_work_item`. If no active one fits → ask the user whether to create a new Epic (workflow `create-epic.md`) or leave it orphaned.

### 2. Gather inputs

| Field | Required | Default |
|-------|----------|---------|
| Title (English) | yes | — |
| Parent Epic ID | recommended | no parent if the user confirms it |
| Area Path | yes | parent Epic's area, or `default_area_path` |
| Description (Goal + Scope) | yes | — |
| Acceptance criteria (high-level) | recommended | — |
| Tags | optional | — |

### 3. `System.Description` template

```markdown
## Goal
{one clear sentence about what the feature delivers}

## Scope
- {item}
- {item}

## High-level acceptance criteria
- [ ] {observable criterion}
- [ ] {observable criterion}

## Out of scope
- {item}
```

### 4. Mandatory preview

```
[project: <alias> — <org>/<project>]
FEATURE (new) → Epic #{parentId} {parentTitle}
└── Title: {title}
    Area: {area}
    AC: {n} bullets
```
Wait for confirmation.

### 5. Create in ADO

`wit_create_work_item` with `workItemType` = `feature` type from config (usually `Feature`).

After creation, **link to the parent Epic**:
`wit_work_items_link` with:
- `sourceId`: id of the newly created Feature
- `targetId`: id of the Epic
- `linkType`: `System.LinkTypes.Hierarchy-Reverse` (the Feature is a child of the Epic)

### 6. Post-create

- Report ID/URL.
- Update `<project_dir>/epics-tree.md` adding the new Feature under the correct Epic, add its row to `<project_dir>/backlog-snapshot.md`, and **bump the snapshot frontmatter's `last_updated` to now (ISO 8601)**.
