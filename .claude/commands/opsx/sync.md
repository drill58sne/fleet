# OpSx Sync Command

Synchronize Fleet configuration state between environments or between local files and a running Fleet instance.

## Usage

```
/opsx sync [source] [target] [flags]
```

## Description

The `sync` command compares and reconciles Fleet configuration between two sources. It supports syncing from local YAML/JSON files to a Fleet instance, between two Fleet instances, or from a Fleet instance back to local files (pull mode).

This command is safe by default — it will preview changes before applying them unless `--auto-approve` is specified.

## Arguments

| Argument | Description |
|----------|-------------|
| `source` | Source of truth (local path, environment name, or Fleet URL) |
| `target` | Destination to sync into (environment name or Fleet URL) |

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--dry-run` | `true` | Preview changes without applying them |
| `--auto-approve` | `false` | Skip confirmation prompt and apply immediately |
| `--pull` | `false` | Reverse sync: pull from target into source (local files) |
| `--filter` | `""` | Comma-separated list of resource types to sync (e.g. `policies,queries,teams`) |
| `--exclude` | `""` | Comma-separated list of resource types to exclude from sync |
| `--conflict` | `ask` | Conflict resolution strategy: `ask`, `source-wins`, `target-wins`, `skip` |
| `--output` | `table` | Output format: `table`, `json`, `yaml` |
| `--context` | `default` | Fleet CLI context/environment to use |

## Supported Resource Types

- `policies` — Fleet policies
- `queries` — Saved queries
- `teams` — Team configurations
- `agent_options` — Agent option profiles
- `labels` — Custom labels
- `enroll_secrets` — Enrollment secrets (redacted in output)
- `packs` — Query packs (legacy)

## Examples

### Preview sync from local files to staging

```bash
/opsx sync ./configs/fleet staging
```

Shows a diff of what would change without applying anything.

### Apply local config to production with confirmation

```bash
/opsx sync ./configs/fleet production --dry-run=false
```

### Auto-approve sync in CI pipeline

```bash
/opsx sync ./configs/fleet production --auto-approve --filter=policies,queries
```

### Pull current state from production to local files

```bash
/opsx sync ./configs/fleet production --pull
```

Useful for bootstrapping a local config directory from an existing Fleet instance.

### Sync between two environments

```bash
/opsx sync staging production --filter=policies
```

Promotes policies from staging to production.

### Sync with conflict resolution

```bash
/opsx sync ./configs/fleet production --conflict=source-wins
```

## Output

The sync command outputs a summary table of changes:

```
Resource Type    Action     Name                          Status
─────────────────────────────────────────────────────────────────
policies         CREATE     Disk Encryption Enabled       pending
policies         UPDATE     OS Version Check              pending
queries          DELETE     Legacy Inventory Query        pending
teams            UNCHANGED  Engineering                   skipped
```

After confirmation (or with `--auto-approve`), status updates to `applied`, `failed`, or `skipped`.

## Conflict Detection

A conflict occurs when a resource has been modified in both source and target since the last known sync point. Conflicts are highlighted in the preview:

```
⚠ CONFLICT  policies  CIS Benchmark Check  (modified in both source and target)
```

Conflict resolution strategies:
- `ask` — Prompt for each conflict interactively (default)
- `source-wins` — Source always overwrites target
- `target-wins` — Target state is preserved, source change is discarded
- `skip` — Skip conflicting resources entirely

## Related Commands

- [`/opsx diff`](diff.md) — Show differences without sync semantics
- [`/opsx apply`](apply.md) — Apply a specific configuration file
- [`/opsx plan`](plan.md) — Generate an execution plan for changes
- [`/opsx rollback`](rollback.md) — Revert a previous sync operation

## Notes

- Enroll secrets are never synced by default for security reasons. Use `--filter=enroll_secrets` explicitly to include them.
- The `--pull` flag will overwrite local files. Ensure your working directory is committed to version control before pulling.
- Sync operations are logged and can be reviewed with `/opsx status`.
