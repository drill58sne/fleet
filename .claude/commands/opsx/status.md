# OpSx Status Command

Provide a comprehensive status overview of the Fleet deployment, infrastructure components, and operational health.

## Usage

```
/opsx status [--component <name>] [--verbose] [--json]
```

## Arguments

- `--component` (optional): Filter status to a specific component (e.g., `mysql`, `redis`, `fleet`, `nginx`, `osquery`)
- `--verbose` (optional): Include detailed diagnostic information and recent logs
- `--json` (optional): Output status as structured JSON for programmatic consumption

## What This Command Does

This command collects and displays the operational status of all Fleet-related infrastructure components. It is designed for operators who need a quick health snapshot or are troubleshooting an incident.

### Components Checked

1. **Fleet Server**
   - Process health and uptime
   - Active connections
   - License validity and expiration
   - Version information
   - Recent error rate from logs

2. **MySQL / TiDB**
   - Connectivity and replication lag
   - Slow query count (last 15 minutes)
   - Active connections vs. max connections
   - Disk usage on data volume

3. **Redis**
   - Connectivity
   - Memory usage vs. `maxmemory`
   - Eviction count
   - Connected clients

4. **osquery Enrollment**
   - Total enrolled hosts
   - Online hosts (last check-in within 30 minutes)
   - Hosts with failing distributed queries
   - Pending policy violations

5. **Nginx / Load Balancer**
   - Active worker connections
   - Request rate (requests/sec)
   - Upstream health checks

6. **Kubernetes / Docker (if applicable)**
   - Pod/container restarts in the last hour
   - Resource utilization (CPU, memory) vs. limits
   - Pending or crashlooping pods

### Status Levels

| Level | Symbol | Meaning |
|-------|--------|---------|
| OK | ✅ | Component is healthy and operating normally |
| WARN | ⚠️ | Component is functional but attention is recommended |
| CRIT | 🔴 | Component is degraded or unavailable; immediate action required |
| UNKNOWN | ❓ | Status could not be determined (check connectivity or permissions) |

## Example Output

```
Fleet OpSx Status — 2024-01-15 14:32:07 UTC
============================================

✅  Fleet Server       v4.42.0   uptime=12d 4h   errors/min=0.02
✅  MySQL              8.0.32    lag=0ms          connections=45/500
⚠️  Redis              7.2.1     memory=87%       evictions=142
✅  osquery Hosts      online=8,421/10,003 (84%)  policy_failures=17
✅  Nginx              req/s=312                  upstreams=3/3 healthy
🔴  k8s fleet-worker   restarts=5 (last 1h)       OOMKilled

Overall: DEGRADED — 1 critical issue, 1 warning
Run `/opsx status --component fleet-worker --verbose` for details.
```

## Step-by-Step Instructions

1. **Gather context** from the user about which environment to check (production, staging, specific cluster/namespace). If not specified, assume production.

2. **Identify available tooling** in the current environment:
   - `kubectl` for Kubernetes deployments
   - `docker` / `docker-compose` for containerized local/staging setups
   - Direct SSH access for bare-metal deployments
   - Fleet API (`fleetctl`) for application-level status

3. **Run health checks** for each component. Use the least-invasive method first (API calls before log parsing before process inspection).

4. **Correlate findings** — if multiple components show issues simultaneously, note potential cascading failures (e.g., Redis memory pressure causing Fleet API slowdowns).

5. **Summarize clearly** with:
   - Overall health verdict
   - Ordered list of issues by severity (CRIT first)
   - Suggested next commands (e.g., `/opsx explore` for deeper investigation, `/opsx apply` for known fixes)

6. **If `--json` is requested**, output a structured object:
   ```json
   {
     "timestamp": "2024-01-15T14:32:07Z",
     "overall": "DEGRADED",
     "components": [
       { "name": "fleet", "status": "OK", "version": "4.42.0", "details": {} },
       { "name": "redis", "status": "WARN", "details": { "memory_pct": 87 } }
     ]
   }
   ```

## Important Notes

- **Read-only**: This command only reads state; it does not modify anything. Safe to run at any time.
- **Permissions**: Requires read access to Fleet API, database metrics endpoint, and cluster/host monitoring. If access is restricted, report UNKNOWN with a clear message rather than failing silently.
- **Thresholds**: WARN threshold for Redis memory is 80%; CRIT is 95%. MySQL connection WARN is 70% of max; CRIT is 90%.
- **osquery online threshold**: A host is considered "online" if it checked in within the last 30 minutes (configurable via `FLEET_OSQUERY_ONLINE_WINDOW`).
