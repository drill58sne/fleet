# opsx logs

Stream and query operational logs from Fleet infrastructure components.

## Usage

```
/opsx logs [component] [options]
```

## Arguments

- `component` — Target component to fetch logs from. One of:
  - `server` — Fleet server application logs
  - `mysql` — MySQL database logs
  - `redis` — Redis cache logs
  - `nginx` — Nginx reverse proxy logs
  - `osquery` — osquery result/status logs
  - `all` — Aggregate logs from all components

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `--tail N` | `100` | Show last N lines |
| `--follow` / `-f` | `false` | Stream logs in real-time |
| `--since DURATION` | `1h` | Show logs from the last duration (e.g. `30m`, `2h`, `1d`) |
| `--level LEVEL` | `info` | Minimum log level: `debug`, `info`, `warn`, `error` |
| `--env ENV` | current | Target environment (`staging`, `production`) |
| `--grep PATTERN` | — | Filter log lines matching a regex pattern |
| `--json` | `false` | Output raw JSON log entries |
| `--no-color` | `false` | Disable colored output |

## Behavior

1. **Resolve environment** — Determine the target environment from `--env` flag or active context set by `/opsx apply`.
2. **Authenticate** — Use stored credentials/kubeconfig to connect to the log source (Kubernetes pod logs, CloudWatch, or direct SSH).
3. **Fetch logs** — Pull log lines according to `--tail` and `--since` constraints.
4. **Filter** — Apply `--level` filtering and `--grep` pattern matching client-side.
5. **Format** — Pretty-print timestamps, levels, and messages unless `--json` is set.
6. **Stream** — If `--follow` is set, maintain the connection and stream new lines until interrupted (Ctrl+C).

## Examples

```bash
# Show last 200 lines from the Fleet server in production
/opsx logs server --tail 200 --env production

# Stream error logs from all components in staging
/opsx logs all --follow --level error --env staging

# Find logs mentioning a specific host UUID
/opsx logs osquery --since 2h --grep "abc123-host-uuid"

# Export raw JSON logs for the last 30 minutes
/opsx logs server --since 30m --json > fleet-server.jsonl

# Watch nginx access logs in real-time
/opsx logs nginx --follow --no-color
```

## Output Format

Default (pretty-printed):
```
2024-01-15 14:32:01 [INFO]  fleet/server  Starting Fleet server  version=4.42.0 pid=1234
2024-01-15 14:32:05 [WARN]  fleet/server  Slow query detected    duration=2.3s query=list_hosts
2024-01-15 14:32:10 [ERROR] fleet/mysql   Connection pool exhausted  pool_size=25 waiting=3
```

JSON mode (`--json`):
```json
{"ts":"2024-01-15T14:32:01Z","level":"info","component":"fleet/server","msg":"Starting Fleet server","version":"4.42.0"}
```

## Notes

- Log retention depends on the environment configuration. Production logs are typically retained for 30 days.
- When using `--follow` with `all`, logs from each component are prefixed with the component name and color-coded.
- Large `--since` windows combined with `--follow` may cause high memory usage; prefer shorter windows for streaming.
- If a component is not running, the command will report its last known status and exit with code 1.
