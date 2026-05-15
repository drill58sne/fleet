# Validate Command

Validate Fleet configuration, policies, and infrastructure state against expected definitions without making any changes.

## Usage

```
/opsx:validate [target] [--strict] [--format=<format>]
```

## Arguments

- `target` — Optional. Specific resource type or name to validate (e.g., `policies`, `queries`, `config`, `all`). Defaults to `all`.
- `--strict` — Treat warnings as errors. Exits with non-zero status if any warnings are found.
- `--format` — Output format: `text` (default), `json`, or `summary`.

## What This Command Does

This command performs read-only validation of your Fleet environment by:

1. **Loading local definitions** from the working directory (YAML/JSON config files, policy files, query packs)
2. **Fetching current remote state** from the Fleet API
3. **Running validation checks** against schema, business rules, and drift detection
4. **Reporting issues** grouped by severity: errors, warnings, and informational notices

No changes are made to the Fleet instance during validation.

## Validation Checks Performed

### Configuration Validation
- YAML/JSON syntax and schema conformance
- Required fields presence and type correctness
- Enum values within allowed ranges
- Cross-reference integrity (e.g., team names referenced in policies must exist)

### Policy Validation
- SQL syntax check for policy queries
- Platform compatibility (macOS/Windows/Linux targets)
- Duplicate policy names within same scope
- Resolution steps present for failing policies

### Query Validation
- osquery SQL syntax validation
- Interval values within acceptable bounds (min: 30s, max: 24h)
- Logging type compatibility with query structure
- Deprecated osquery table usage warnings

### Drift Detection
- Local definitions vs. current remote state comparison
- Identifies resources that exist remotely but not locally (orphaned)
- Identifies local resources not yet applied remotely
- Flags resources where remote state differs from local definition

### Security Checks
- Policies without resolution steps flagged as warnings
- Overly broad queries (missing WHERE clauses on large tables)
- Sensitive data exposure in query results (e.g., plaintext passwords in query columns)

## Output

### Text Format (default)

```
Validating Fleet configuration...

✓ Config schema: valid
✓ Teams: 3 defined, 3 found remotely
⚠ Policies: 12 defined
  WARNING  policy "Ensure FileVault enabled" missing resolution steps
  WARNING  policy "Check SSH config" targets deprecated table 'ssh_configs'
✗ Queries: 8 defined
  ERROR    query "Get all processes" interval 10 is below minimum (30)
  ERROR    query "Kernel extensions" references unknown table 'kernel_extensions' on Windows

Summary: 2 errors, 2 warnings, 0 notices
Validation FAILED
```

### JSON Format

```json
{
  "status": "failed",
  "errors": 2,
  "warnings": 2,
  "notices": 0,
  "results": [
    {
      "resource": "query",
      "name": "Get all processes",
      "severity": "error",
      "code": "INTERVAL_TOO_LOW",
      "message": "interval 10 is below minimum allowed value of 30"
    }
  ]
}
```

### Summary Format

One-line output suitable for CI status checks:

```
FAILED: 2 errors, 2 warnings across 23 resources
```

## Exit Codes

| Code | Meaning |
|------|---------|
| `0`  | Validation passed (no errors) |
| `1`  | Validation failed (one or more errors) |
| `2`  | Validation passed but warnings found (only with `--strict`) |
| `3`  | Unable to connect to Fleet API or load local files |

## Examples

```bash
# Validate everything
/opsx:validate

# Validate only policies
/opsx:validate policies

# Strict mode — fail on warnings too (useful in CI)
/opsx:validate --strict

# JSON output for programmatic consumption
/opsx:validate --format=json

# Validate a specific named resource
/opsx:validate queries/endpoint-security
```

## Integration with Other Commands

- Run **validate** before **plan** to catch schema errors early
- Run **validate** after **apply** to confirm state matches expectations
- Use in CI pipelines as a pre-merge check on pull requests that modify Fleet configs
- Combine with **diff** to understand both what changed and whether those changes are valid

## Notes

- Validation requires read access to the Fleet API (uses the same credentials as other opsx commands)
- SQL syntax validation is performed locally using an embedded osquery schema; it does not execute queries
- The `--strict` flag is recommended for CI/CD pipelines to enforce policy hygiene
