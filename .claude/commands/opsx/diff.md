# opsx diff

Analyze and summarize differences between Fleet configurations, database migrations, or code changes.

## Usage

```
/opsx diff [source] [target] [--type <config|migration|schema|api>]
```

## Arguments

- `source` — Base reference (branch name, commit SHA, file path, or "current")
- `target` — Target reference to compare against (branch name, commit SHA, file path, or "HEAD")
- `--type` — Optional filter to focus diff analysis on a specific area

## What This Command Does

This command helps you understand the impact of changes in the Fleet codebase by:

1. **Identifying changed files** relevant to the specified type
2. **Summarizing migration changes** — new tables, altered columns, dropped indexes
3. **Highlighting API surface changes** — new endpoints, modified request/response shapes, deprecated routes
4. **Flagging breaking changes** — anything that could affect existing deployments or integrations
5. **Checking config drift** — differences in default values, new required fields, removed options

## Steps

### 1. Gather Diff Context

If `source` and `target` are git references:
- Run `git diff <source>..<target> --name-only` to list changed files
- Filter by `--type` if provided
- For each relevant file, fetch the full diff

If paths are provided:
- Read both files and compute a structured diff

### 2. Categorize Changes

Group changes into:
- **Database migrations** (`db/migrations/`, `server/datastore/mysql/migrations/`)
- **API handlers** (`server/service/`, `server/fleet/`)
- **Configuration** (`config/`, `*.yml`, `*.yaml`, `*.json` config files)
- **Frontend** (`frontend/`, `*.tsx`, `*.ts`)
- **Deployment** (`charts/`, `Dockerfile`, `docker-compose*`)

### 3. Analyze Each Category

#### Database Migrations
- List migration files added or modified
- Summarize schema changes (ADD COLUMN, DROP TABLE, CREATE INDEX, etc.)
- Flag irreversible operations (DROP, truncation)
- Check if down migrations exist and are safe

#### API Changes
- Identify new, modified, or removed route handlers
- Note changes to request/response structs in `server/fleet/`
- Flag any removed or renamed fields (potential breaking change)
- Check version compatibility (are old clients still supported?)

#### Configuration Changes
- List new config keys and their defaults
- Flag removed or renamed config keys
- Note any keys that changed from optional to required

### 4. Produce Summary Report

Output a structured summary:

```
## Diff Summary: <source> → <target>

### ⚠️  Breaking Changes
- <list any breaking changes>

### 🗄️  Database Migrations (<count> files)
- <migration name>: <description>

### 🌐 API Changes (<count> endpoints)
- [NEW] POST /api/v1/fleet/...
- [MODIFIED] GET /api/v1/fleet/... — added field `foo`
- [REMOVED] DELETE /api/v1/fleet/... ⚠️

### ⚙️  Configuration Changes
- [NEW] server.new_option (default: false)
- [REMOVED] deprecated.old_option ⚠️

### 📦 Other Changes
- <count> frontend files
- <count> test files
- <count> documentation files
```

### 5. Upgrade Guidance

If migrations or breaking API changes are detected, append:

```
## Upgrade Notes

Before deploying this change:
1. Back up the database
2. Run migrations: `fleet prepare db`
3. Update any integrations using removed/changed endpoints
4. Review new config options and set values appropriate for your environment
```

## Examples

```
/opsx diff main HEAD
/opsx diff v4.50.0 v4.51.0 --type migration
/opsx diff current HEAD --type api
/opsx diff ./config/old.yml ./config/new.yml --type config
```

## Notes

- Always flag DROP or destructive SQL operations as high-risk
- If a migration has no corresponding down migration, note it explicitly
- Cross-reference API changes with the Fleet API documentation when available
- For large diffs (>50 files), summarize by directory rather than individual files
