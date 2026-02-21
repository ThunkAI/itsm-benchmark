# Diagnosis to Action Guide

Quick reference for mapping diagnosis categories to recommended service actions.

## Resource Exhaustion

| Diagnosis                   | Recommended Action                                           | Rationale                                                                                                                                  |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `cpu_saturation`            | `service-scale` OR escalate (see below)                      | **Application services**: Scale to distribute load. **Databases**: Escalate to DBA for query optimization, index creation, or tuning.      |
| `memory_leak`               | `service-restart` then `service-revert-deployment` if recurs | Restart clears memory; revert if bug in current version                                                                                    |
| `disk_full`                 | `service-restart` OR `service-clear-database-logs`           | **Application services** (ephemeral storage): restart clears container disk. **Databases** (persistent storage): clear logs to free space. |
| `connection_pool_exhausted` | `service-restart`                                            | Reset connection pools and flush zombie connections                                                                                        |
| `thread_pool_exhausted`     | `service-scale`                                              | Add replicas to handle request concurrency                                                                                                 |

## Configuration Issues

| Diagnosis                      | Recommended Action       | Rationale                                                                      |
| ------------------------------ | ------------------------ | ------------------------------------------------------------------------------ |
| `bad_configuration`            | Contextual - see below   | Requires careful analysis to determine the source of the configuration change. |
| `missing_environment_variable` | `service-update-env-var` | Set the missing environment variable                                           |

## Network & Connectivity

| Diagnosis                    | Recommended Action           | Rationale                                            |
| ---------------------------- | ---------------------------- | ---------------------------------------------------- |
| `network_connectivity_issue` | `service-restart`            | Reset network connections                            |
| `network_mesh_latency`       | Manual intervention required | Requires service mesh/platform configuration changes |
| `ddos_attack`                | `service-block-ip-address`   | Block malicious IP addresses at firewall/WAF         |
| `rate_limit_exceeded`        | `service-scale` or wait      | Scale to distribute requests or wait for quota reset |
| `dns_resolution_failure`     | `service-restart`            | Refresh DNS cache                                    |

## Cache & Data

| Diagnosis           | Recommended Action          | Rationale                                        |
| ------------------- | --------------------------- | ------------------------------------------------ |
| `stale_cache`       | `service-flush-cache`       | Flush cached data to force refresh               |
| `data_corruption`   | `service-set-power-state`   | Stop corrupted service to prevent further damage |
| `schema_mismatch`   | `service-revert-deployment` | Rollback incompatible code version               |
| `database_deadlock` | `service-restart`           | Reset connection pools and transactions          |

## Service State

| Diagnosis               | Recommended Action              | Rationale                            |
| ----------------------- | ------------------------------- | ------------------------------------ |
| `service_crash`         | `service-restart`               | Restore service to running state     |
| `circuit_breaker_state` | `service-reset-circuit-breaker` | Manually reset stuck circuit breaker |

## Security

| Diagnosis             | Recommended Action           | Rationale                         |
| --------------------- | ---------------------------- | --------------------------------- |
| `expired_certificate` | `service-renew-certificate`  | Renew expired SSL/TLS certificate |
| `invalid_credentials` | Manual intervention required | Requires credential rotation      |

## Special Cases

| Diagnosis        | Recommended Action | Rationale                                  |
| ---------------- | ------------------ | ------------------------------------------ |
| `healthy_system` | None               | No action required                         |
| `undiagnosable`  | Escalate to human  | Insufficient data for automated resolution |

## Bad Configuration Decision Tree

**`bad_configuration`** requires careful analysis to determine the correct action. Follow this decision tree:

### Step 1: Identify the Configuration Change Source

Check logs and service info to determine **what changed** and **when**:

1. **Feature Flag Change**

   - Log entry shows: `"Feature flag [NAME] is now ON/OFF"` or similar
   - Action timing correlates with incident start time
   - **Correct Action**: `service-set-feature-flag` (toggle the specific flag)
   - **If tool unavailable**: Escalate to engineering team

2. **Recent Deployment** (within hours/days of incident)

   - Service info shows `last_deployed` timestamp correlates with incident start
   - Logs show deployment or config reload messages at incident start time
   - **Correct Action**: `service-revert-deployment` (rollback the deployment)

3. **Manual Configuration Change** (config file edit, parameter update)
   - Logs show: `"Configuration reloaded"` or similar
   - No recent deployment or feature flag change
   - **Correct Action**: `service-revert-deployment` OR escalate if unclear

### Step 2: Temporal Analysis

**Critical Rule**: Only revert a deployment if the deployment timing correlates with the incident.

- ✅ Deployment at `14:30`, incident started `14:35` → Revert deployment
- ❌ Deployment `7 days ago`, incident started `30 minutes ago` → DO NOT revert deployment
- ✅ Feature flag enabled `30 minutes ago`, incident started `30 minutes ago` → Toggle feature flag

### Step 3: Tool Availability Check

If the required tool is not available:

1. Document what action is needed (e.g., "Feature flag X should be disabled")
2. Escalate to the appropriate engineering team
3. Explain why the specific action is required

### Examples

**Example 1: Feature Flag Issue**

- Logs: `"{{NOW-30m}} Feature flag ENABLE_NEW_FLOW is now ON"`
- Service deployed: 7 days ago
- Incident started: 30 minutes ago
- **Action**: Use `service-set-feature-flag` to disable flag (if available), otherwise escalate

**Example 2: Bad Deployment**

- Service deployed: 2 hours ago
- Incident started: 2 hours ago
- Logs show new errors after deployment
- **Action**: `service-revert-deployment`

**Example 3: Unknown Config Source**

- Logs show config errors but no clear source
- No recent deployment, no feature flag changes
- **Action**: Escalate to engineering team for investigation

## Database CPU Saturation - Special Handling

When diagnosing **`cpu_saturation`** on a **database service** (e.g., `primary-db`, `inventory-db`), follow these rules:

### Identification

Database CPU saturation is typically caused by:

1. **Missing indexes** - Queries performing full table scans
2. **Inefficient queries** - Poorly optimized SQL causing excessive CPU usage
3. **Query storms** - Sudden spike in query volume
4. **Lock contention** - Queries waiting on locks, consuming CPU while waiting

### Evidence in Logs

Look for these patterns:

- `"slow query"`, `"query exceeded 1000ms"`
- `"full table scan"`, `"missing index"`
- `"CPU usage high: 95%"` combined with slow query logs
- High database latency metrics (e.g., 5ms → 8000ms)

### CRITICAL: Do NOT Revert Database Deployments

❌ **WRONG ACTION**: `service-revert-deployment` on a database

- Database "deployments" are typically years old
- Reverting a database is dangerous and rarely the solution
- Database performance issues require DBA expertise, not deployment rollbacks

✅ **CORRECT ACTION**: Escalate to Database_Administration team

**Reasoning:**

1. Missing indexes require `CREATE INDEX` statements (DBA task)
2. Slow queries may need query rewriting (application code change)
3. Database tuning requires expert knowledge
4. You cannot fix database performance with service actions

### Decision Tree for Database CPU Saturation

1. **Identify affected service**: Database service (e.g., `primary-db`)
2. **Check logs**: Look for "slow query", "missing index", "full table scan"
3. **Verify CPU metrics**: Database CPU at 90%+ with high query latency
4. **Action**: Set `action_solution: null` and escalate to Database_Administration
5. **Escalation reason**: Include details about slow queries, missing indexes, and why DBA intervention is needed

### Example

**Scenario**: Database logs show "missing index on orders(status, created_at)" and CPU at 95%

```json
{
  "diagnosis_category": "cpu_saturation",
  "affected_service": "primary-db",
  "action_solution": null,
  "ticket_outcome": {
    "tool": "ticket-escalate",
    "assignment_group": "Database_Administration",
    "reason": "Primary-db CPU saturation from slow queries. Missing index on orders table causing full table scans. Requires DBA intervention for index creation."
  }
}
```

## Tool Precedence Rules

When multiple actions could apply:

1. **Scale before restart** - For load-related issues (`cpu_saturation`, `thread_pool_exhausted`), prefer `service-scale` over `service-restart`
2. **Restart before revert** - Try `service-restart` before `service-revert-deployment` unless deployment is clearly the cause
3. **Revert over manual** - Use `service-revert-deployment` if issue started after recent deployment
4. **Temporal correlation required** - Only revert deployment if deployment timestamp correlates with incident start time
