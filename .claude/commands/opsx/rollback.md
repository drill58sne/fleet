# OpsX Rollback Command

Rollback a Fleet deployment or configuration change to a previous known-good state.

## Usage

```
/opsx rollback [target] [--to <version|timestamp|tag>] [--dry-run] [--force]
```

## Arguments

- `target` — The deployment target or component to rollback (e.g., `fleet-server`, `fleet-worker`, `config`, `migrations`)
- `--to` — Specific version, timestamp (ISO 8601), or tag to roll back to. Defaults to the previous stable state.
- `--dry-run` — Preview what would be rolled back without making changes
- `--force` — Skip confirmation prompts (use with caution in production)

## Behavior

When invoked, this command will:

1. **Identify the current state** — Query the deployment registry or Kubernetes cluster to determine the currently running version and configuration.

2. **Determine rollback target** — If `--to` is not specified, identify the last known-good deployment from the deployment history. If `--to` is specified, validate that the target version/timestamp exists in the registry.

3. **Pre-flight checks** — Before executing the rollback:
   - Verify the target state artifacts are still available (container images, config snapshots)
   - Check for any database migration incompatibilities (downward migrations may require manual intervention)
   - Warn if the rollback would affect more than one major version
   - Check for any active user sessions or in-flight jobs that may be disrupted

4. **Execute rollback** — Depending on the target:
   - **fleet-server / fleet-worker**: Update the Kubernetes deployment image tag and rollout
   - **config**: Restore the previous ConfigMap or environment variable set from the snapshot store
   - **migrations**: Emit a warning — database migration rollbacks require manual SQL execution; provide the rollback SQL if available

5. **Verify rollback** — After applying changes:
   - Wait for the rollout to complete (or timeout after 5 minutes)
   - Run a basic health check against the `/healthz` endpoint
   - Report the final running version

6. **Audit log** — Record the rollback action, actor, timestamp, and from/to versions in the ops audit log.

## Examples

### Roll back Fleet server to the previous deployment
```
/opsx rollback fleet-server
```

### Roll back to a specific version
```
/opsx rollback fleet-server --to v4.52.1
```

### Roll back configuration to a snapshot from yesterday
```
/opsx rollback config --to 2024-11-14T09:00:00Z
```

### Preview what a rollback would do
```
/opsx rollback fleet-server --dry-run
```

### Force rollback without confirmation (CI/automation)
```
/opsx rollback fleet-server --to v4.51.0 --force
```

## Output Format

The command outputs a structured summary:

```
Rollback Plan
=============
Target:       fleet-server
Current:      v4.53.0 (deployed 2024-11-15T14:32:00Z by alice)
Rolling back: v4.52.1 (deployed 2024-11-14T09:15:00Z by bob)

Changes:
  - Image: ghcr.io/fleetdm/fleet:v4.53.0 → ghcr.io/fleetdm/fleet:v4.52.1
  - No config changes detected
  - No migration rollback required

Pre-flight: ✓ Image available  ✓ No breaking migration  ✓ No active MDM enrollments in progress

Proceed with rollback? [y/N]
```

## Safety Notes

- **Database migrations** are not automatically reversed. If the rollback spans a schema migration, the command will halt and provide instructions for manual intervention.
- Rolling back across more than **2 minor versions** will trigger an additional confirmation step.
- In production environments, prefer using `--dry-run` first to review the impact.
- This command does **not** roll back osquery configuration or MDM profiles pushed to enrolled hosts.

## Related Commands

- `/opsx apply` — Apply a new deployment or configuration
- `/opsx status` — Check current deployment status
- `/opsx diff` — Compare two versions or states
- `/opsx archive` — Archive and snapshot the current state before making changes
