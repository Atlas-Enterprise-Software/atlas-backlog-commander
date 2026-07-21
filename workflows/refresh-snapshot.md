# Workflow — Refresh backlog snapshot

> Multi-project context: see `CLAUDE.md → Workflow execution`. Relevant config: the **Process** section (`work_item_types`, `states`) and `wiql_base_filter`.

Procedure Claude runs when the user says "refresh the snapshot", "update the backlog" or similar.

## Process placeholders (from `config.md` → `## Process`)

Placeholder names match the config keys exactly (see `CLAUDE.md → Process per project`):

- `<epic>`, `<feature>`, `<requirement>`, `<bug>`: actual type names (e.g. Scrum: `Product Backlog Item`; Agile: `User Story`; Basic: `Issue`, no `Feature`).
- `<closed_states>`: states that take an item out of the active backlog (e.g. `'Done','Closed'`).
- `<removed_states>`: usually `'Removed'`.
- `<inactive_parent_states>`: states that disqualify an item as a parent (golden rule 5) — used in the merge's parent validation.
- If `feature` is empty (a process without that level, e.g. Basic) → omit its section/queries and the tree becomes 2 levels (`<epic>` → `<requirement>`/`<bug>`).

## Mode selection

`<project_dir>/backlog-snapshot.md` carries YAML frontmatter with the last update date:

```markdown
---
last_updated: 2026-07-14T10:30:00+02:00
last_full_refresh: 2026-07-14
---
# Backlog Snapshot — <alias> — 2026-07-14 10:30
```

- **Incremental (default)**: if frontmatter with `last_updated` exists → run the "Incremental mode" steps. Only items created/modified after that date are queried.
- **Full**: run the 7 steps of "Full mode" when: there is no frontmatter (migration / first time), the user explicitly asks for a "full refresh", or `last_full_refresh` is more than 30 days old.

## Incremental mode

### I1. Read base date
- Load `<project_dir>/config.md` (`organization`, `project`, Process section) and the `last_updated` from the snapshot frontmatter.

### I2. Delta WIQL query (a single one)

Covers items **created and modified** (for a newly created item, `ChangedDate >= CreatedDate` always holds). Use only the **date part** of `last_updated` (WIQL compares at day precision); the same-day overlap is resolved in the merge by ID.

```sql
SELECT [System.Id]
FROM workitems
WHERE [System.TeamProject] = '<project>'
  AND [System.WorkItemType] IN ('<epic>','<feature>','<requirement>','<bug>')
  AND [System.ChangedDate] >= '<YYYY-MM-DD of last_updated>'
ORDER BY [System.ChangedDate] DESC
```

If it returns no IDs → update `last_updated` and report "no changes".

### I3. Batch fetch
`wit_get_work_items_batch_by_ids` with the standard fields (`System.Id, System.WorkItemType, System.State, System.AreaPath, System.Title, System.Tags, System.Parent, System.ChangedDate`). Batches of 200.

### I4. Merge onto the existing snapshot (row-level Edit, not a full rewrite)

- **New ID** → add a row in its section of `backlog-snapshot.md` + a branch in `epics-tree.md`.
- **Existing ID** → update State/Area/Parent/Title/Tags of its row (and its position in the tree if the parent changed).
- **Moves to `<closed_states>` or `<removed_states>`** → remove from the active section and add to "Recently Closed (30d)".
- **Parent validation**: for each new/updated item, check that its `System.Parent` exists in the snapshot. If it does not (parent created/moved outside the delta), call `wit_get_work_items_batch_by_ids` with the missing parents and add them to their section and to the tree. If the parent is in `<inactive_parent_states>` (config), flag the item in the report as **"inactive parent"** (re-parenting candidate, golden rule 5).
- Do **not** recompute the totals line: it is labeled `as of last full refresh (<date>)` and only Full mode rewrites it (recounting the whole file on every delta is not worth the tokens).

### I5. Update dates
- Frontmatter `last_updated` = now (ISO 8601 with time). Also update the date in the title. `last_full_refresh` is not touched.

### I6. Report the delta
Brief summary: N new, M updated, K closed, parents added/inactive. **No table dumps** in the chat.

## Full mode

### 1. Read config
- Load `<project_dir>/config.md` to get `organization`, `project`, `wiql_base_filter` and the Process section.

### 2. Run 4 WIQL queries

Use `wit_query_by_wiql`. Fields to always return:
`System.Id, System.WorkItemType, System.State, System.AreaPath, System.Title, System.Tags, System.Parent, System.ChangedDate`.

**Query A — Epics and Features** (omit `<feature>` if the process does not have it):
```sql
SELECT [System.Id]
FROM workitems
WHERE [System.TeamProject] = '<project>'
  AND [System.WorkItemType] IN ('<epic>','<feature>')
  AND [System.State] NOT IN (<removed_states>,<closed_states>)
ORDER BY [System.WorkItemType], [System.Id]
```

**Query B — Active backlog items (`<requirement>`):**
```sql
SELECT [System.Id]
FROM workitems
WHERE [System.TeamProject] = '<project>'
  AND [System.WorkItemType] = '<requirement>'
  AND [System.State] NOT IN (<removed_states>,<closed_states>)
ORDER BY [System.AreaPath], [System.Id]
```

**Query C — Active Bugs** (if the process has a `<bug>` type):
```sql
SELECT [System.Id]
FROM workitems
WHERE [System.TeamProject] = '<project>'
  AND [System.WorkItemType] = '<bug>'
  AND [System.State] NOT IN (<removed_states>,<closed_states>)
ORDER BY [Microsoft.VSTS.Common.Severity], [System.Id]
```

**Query D — Recently closed (last 30 days, top 50):**
```sql
SELECT [System.Id]
FROM workitems
WHERE [System.TeamProject] = '<project>'
  AND [System.WorkItemType] IN ('<requirement>','<bug>','<feature>')
  AND [System.State] IN (<closed_states>)
  AND [System.ChangedDate] >= @Today - 30
ORDER BY [System.ChangedDate] DESC
```

### 3. Batch fetch details

For each returned ID list, call `wit_get_work_items_batch_by_ids` with the minimal fields above. If > 200 ids, split into batches.

### 4. Render `<project_dir>/backlog-snapshot.md`

Exact structure (compact format, no descriptions). Section headings use the project's actual type name (e.g. `## Active PBIs` in cb, `## Active User Stories` in an Agile project). The frontmatter is mandatory: `last_updated` = now (ISO 8601), `last_full_refresh` = today.

```markdown
---
last_updated: YYYY-MM-DDTHH:MM:SS+02:00
last_full_refresh: YYYY-MM-DD
---
# Backlog Snapshot — <alias> — YYYY-MM-DD HH:MM

> Totals as of last full refresh (YYYY-MM-DD): N Epics · N Features · N <requirement plural> · N Bugs. Incremental merges do not update this line.

## Epics
| ID | State | Area | Title |
|----|-------|------|-------|
| 1234 | Active | Area\Sub | Patient MedicalRecords v2 |

## Features
| ID | State | Area | Parent | Title |
|----|-------|------|--------|-------|
| 1245 | Committed | ...\Sub | 1234 | Simulation engine |

## Active <requirement plural>
| ID | State | Area | Parent | Title | Tags |
|----|-------|------|--------|-------|------|

## Active Bugs
| ID | State | Area | Sev | Title | Tags |
|----|-------|------|-----|-------|------|

## Recently Closed (30d)
| ID | Type | Closed | Title |
|----|------|--------|-------|
```

### 5. Render `<project_dir>/epics-tree.md`

Rebuild the hierarchy from `System.Parent` according to the config's `backlog_levels` (typical: `<feature>`→`<epic>` and `<requirement>`/`<bug>`→`<feature>`; in processes without `<feature>`, 2 levels):

```
EPIC #1234 — Patient MedicalRecords v2 (Active)
├── FEATURE #1245 — Simulation engine (Committed) — 8 PBIs (3 Done, 5 Open)
│   ├── PBI #35047 — Add scenario picker (Active)
│   └── PBI #35048 — Persist simulation snapshot (New)
└── FEATURE #1278 — Reporting export (Active) — 4 PBIs
```

### 6. Historical snapshot (optional)

Copy `backlog-snapshot.md` to `<project_dir>/history/snapshot-YYYY-MM-DD.md` (file name = the snapshot's `last_full_refresh` date). Pruning: keep only the last 3 snapshots.

### 7. Report to the user

Brief summary: totals per type, items created since the last snapshot, items moved to closed. **Do not paste the whole table** in the chat — the user already has it on disk.
