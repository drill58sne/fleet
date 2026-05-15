# OpsX Plan Command

Generate an execution plan for infrastructure changes before applying them. Shows what will be created, modified, or destroyed without making any actual changes.

## Usage

```
/opsx plan [environment] [--module <module>] [--var-file <file>] [--out <planfile>]
```

## Arguments

- `environment` — Target environment (e.g., `production`, `staging`, `dev`). Defaults to `staging`.
- `--module` — Limit planning to a specific Terraform module or service.
- `--var-file` — Path to a `.tfvars` file to use for variable overrides.
- `--out` — Save the plan to a file for later use with `/opsx apply`.

## What This Command Does

1. **Validates** the current Terraform configuration for syntax errors.
2. **Refreshes** remote state to detect any drift between actual infrastructure and recorded state.
3. **Computes** a diff of what changes would be applied.
4. **Summarizes** additions, modifications, and destructions in a human-readable format.
5. **Flags** any high-risk operations (e.g., destroying databases, replacing EC2 instances).

## Example Output

```
Planning changes for: staging
Module: fleet-core

Refreshing state... done.

Terraform will perform the following actions:

  # aws_ecs_service.fleet will be updated in-place
  ~ resource "aws_ecs_service" "fleet" {
      ~ task_definition = "fleet:42" -> "fleet:43"
    }

  # aws_elasticache_cluster.redis will be replaced
  -/+ resource "aws_elasticache_cluster" "redis" {
      ~ node_type = "cache.t3.micro" -> "cache.t3.small" # forces replacement
    }

Plan: 1 to add, 1 to change, 1 to destroy.

⚠️  WARNING: 1 resource will be destroyed and recreated.
    This may cause downtime for: aws_elasticache_cluster.redis

To apply this plan, run: /opsx apply staging --plan-file plan.out
```

## Risk Assessment

The plan command automatically categorizes changes by risk level:

| Risk Level | Examples |
|------------|----------|
| 🟢 Low     | Adding new resources, updating tags, scaling up |
| 🟡 Medium  | In-place updates to running services, security group changes |
| 🔴 High    | Destroying databases, replacing stateful resources, VPC changes |

High-risk changes will require explicit confirmation before `/opsx apply` proceeds.

## Saving Plans

Save a plan to ensure that exactly the reviewed changes are applied:

```
/opsx plan production --out prod-plan-2024-01-15.out
```

Then apply the saved plan:

```
/opsx apply production --plan-file prod-plan-2024-01-15.out
```

Saved plans are stored in `.claude/plans/` and are valid for 24 hours. After expiry, you must re-run `/opsx plan` to generate a fresh plan.

## Integration with Other Commands

- Use `/opsx diff` to inspect raw Terraform diffs before planning.
- Use `/opsx status` to check the current state of infrastructure before planning.
- Use `/opsx apply` to execute a saved or fresh plan.
- Use `/opsx rollback` if an applied plan causes unexpected issues.

## Notes

- Planning does **not** acquire a state lock in most backends, but some backends (e.g., S3 + DynamoDB) may briefly lock during state refresh.
- Always run `/opsx plan` before `/opsx apply` in production environments.
- Plans are non-destructive and safe to run at any time.
