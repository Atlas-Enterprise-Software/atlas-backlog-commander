# Workflow — Create Epic

> Multi-project context: see `CLAUDE.md → Workflow execution`. Relevant config: the actual name of the `epic` type.

For strategic Epics / top-level initiatives not covered by `atlas-*` skills.

## Steps

### 1. Pre-checks
1. Check `<project_dir>/epics-tree.md` for existing Epics with overlapping scope — **Grep the `EPIC #` lines by keywords; do not Read the whole tree** (golden rule 2). If there is a match → warn the user and propose extending the existing one instead of creating a new one.
2. Read `<project_dir>/config.md` for `default_area_path`, org/project and the process `epic` type.

### 2. Gather inputs (in a single message if several are missing)

| Field | Required | Default |
|-------|----------|---------|
| Title (English) | yes | — |
| Area Path | yes | `default_area_path` from config |
| Vision (1-2 sentences) | yes | — |
| Goals (3-5 bullets) | yes | — |
| Success metrics | recommended | — |
| In-scope / Out-of-scope | recommended | — |
| Stakeholders | recommended | — |
| Target iteration | optional | unassigned |
| Tags | optional | — |

### 3. `System.Description` structure (Markdown → HTML via MCP)

```markdown
## Vision
{vision}

## Goals
- {goal 1}
- {goal 2}

## Success metrics
- {metric 1}

## In scope
- {item}

## Out of scope
- {item}
```

### 4. Mandatory preview

Show the user a tree preview before creating:
```
[project: <alias> — <org>/<project>]
EPIC (new)
└── Title: {title}
    Area: {area}
    Goals: {n} | Metrics: {n} | Stakeholders: {n}
```
Wait for explicit confirmation ("ok", "create", "go").

### 5. Create in ADO

`wit_create_work_item` with:
- `workItemType`: `epic` type from config (usually `Epic`)
- `fields`:
  - `System.Title`: title
  - `System.AreaPath`: area
  - `System.Description`: HTML from step 3
  - `System.Tags`: tags separated by `;`

### 6. Post-create

- Report the assigned ID and URL to the user.
- If the snapshot exists, manually add the row to `<project_dir>/epics-tree.md` and `<project_dir>/backlog-snapshot.md` (no need to regenerate everything), and **bump the snapshot frontmatter's `last_updated` to now (ISO 8601)**.
