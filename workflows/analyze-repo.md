# Workflow — Analyze Repo (structure, direct reading)

> Multi-project context: see `CLAUDE.md → Workflow execution`.

Ensures a repo is available locally and reads its structure to feed backlog
generation. Reuses what is already cloned; clones only what is missing. **Does not use graphify** (AST-only adds
nothing over a direct read, and the AI mode is ruled out due to cost).

Relevant config in `<project_dir>/config.md` → **Local repositories** section (`repos_root`, `clone_url_pattern`).

## Steps

### 1. Resolve `repos_root` and `clone_url_pattern`

1. Read `repos_root` and `clone_url_pattern` from `<project_dir>/config.md`.
2. If either is undefined → ask the user (for `clone_url_pattern`, propose the standard ADO pattern: `https://<org>@dev.azure.com/<org>/<project url-encoded>/_git/<RepoName>`) and **persist it** in `<project_dir>/config.md`.

### 2. Reuse or clone the repo

1. Resolve the exact `<RepoName>` against `<project_dir>/repos.md` or `repo_list_repos_by_project`.
2. If `<repos_root>/<RepoName>` exists and is a git repo → **reuse** (do not clone). `git fetch`/`pull` only if the user asks for it.
3. If it does not exist → clone using the pattern from the config:
   ```bash
   git clone "<clone_url_pattern with RepoName substituted>" "<repos_root>/<RepoName>"
   ```

### 3. Read the structure (straight from the tree)

On `<repos_root>/<RepoName>`, extract the signals the backlog needs. **Branch by the repo's `tech_stack`** (from `<project_dir>/repos.md`; if unknown, infer it first from the root files: `*.sln`/`*.csproj` → .NET, `package.json` → Node/JS/TS, `host.json` → Azure Functions).

**.NET (Clean Architecture):**
```bash
R="<repos_root>/<RepoName>"
# Layers (projects = .csproj)
find "$R" -name '*.csproj' -not -path '*/obj/*' | sed "s#$R/##" | sort
# Domain entities
find "$R" -path '*Domain*' -name '*.cs' -not -path '*/obj/*' -not -path '*/bin/*' | grep -vi test | xargs -n1 basename | sort -u
# Endpoints / WebApi
find "$R" -path '*WebApi*' \( -name '*Endpoints.cs' -o -name '*Controller.cs' \) -not -path '*/obj/*' | xargs -n1 basename | sort -u
```

**Node / JS / TS:**
```bash
R="<repos_root>/<RepoName>"
# Package layout (monorepo workspaces or single package) + scripts/deps
find "$R" -name package.json -not -path '*/node_modules/*' | sed "s#$R/##"
# Source modules (top 2 levels under src/)
find "$R/src" -maxdepth 2 -type d 2>/dev/null | sed "s#$R/##"
# Routes/handlers by convention
find "$R" -not -path '*/node_modules/*' \( -name '*route*' -o -name '*controller*' -o -name '*handler*' \) -type f | sed "s#$R/##" | sort
```

**Other / unknown stacks:**
```bash
R="<repos_root>/<RepoName>"
# Cheap overview: tracked files grouped by top-level directory
git -C "$R" ls-files | cut -d/ -f1-2 | sort | uniq -c | sort -rn | head -30
```

- For specific detail (what an endpoint does, what fields an entity has) → open the specific file with Read or `repo_get_file_content`.

### 4. Consume

- Use layers + entities + endpoints to feed the backlog items (`atlas-*` skills or workflows) and the repo → area mapping.

## Notes

- Cheap, offline reading; produces no artifacts in the repo or in the workspace.
