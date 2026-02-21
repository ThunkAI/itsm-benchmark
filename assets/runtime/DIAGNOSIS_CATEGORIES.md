# System Diagnosis Categories

This document defines the complete taxonomy of diagnosis categories used to classify what is objectively wrong with a system. These categories represent ground truth about the system state, independent of how users describe issues in tickets.

## Category Definitions

### Healthy System

**`healthy_system`** - System is operating normally with no detectable issues. All services functioning within normal parameters.

### Diagnostic Limitations

**`undiagnosable`** - System has issues but root cause cannot be determined with available observability data.

### Resource Exhaustion

**`cpu_saturation`** - CPU usage at or near 100%, causing performance degradation or service unresponsiveness.

**Common causes:**

1. **High load** - Traffic spike exceeding service capacity from **legitimate users**
2. **Inefficient code** - CPU-intensive operations (e.g., complex algorithms, heavy computation)
3. **Missing database indexes** - Queries performing full table scans instead of index lookups
4. **Resource-intensive queries** - Poorly optimized database queries consuming excessive CPU

**Important distinction - Missing Index is CPU Saturation, NOT Schema Mismatch:**

- ❌ **WRONG**: "Missing index" = `schema_mismatch`
- ✅ **CORRECT**: "Missing index causing slow queries and high CPU" = `cpu_saturation`
- A missing index is a **performance optimization issue**, not a schema compatibility issue
- Schema mismatch means the database structure doesn't match what the application expects (missing columns, wrong types, etc.)
- Missing indexes cause full table scans, leading to high CPU usage and slow query performance

**Important distinction - DDoS Attack causes CPU Saturation, but diagnose as DDoS:**

- ❌ **WRONG**: High CPU from malicious traffic = `cpu_saturation`
- ✅ **CORRECT**: High CPU from malicious traffic = `ddos_attack`
- If traffic spike is from **malicious sources** (single IP, bot user agents, rate limit warnings), diagnose as `ddos_attack`
- If traffic spike is from **legitimate users**, diagnose as `cpu_saturation`
- Check logs for suspicious patterns (repeated requests from same IP, bot user agents) before diagnosing as CPU saturation

**How to identify CPU saturation:**

1. Check CPU metrics showing sustained usage near 100%
2. **Rule out malicious traffic first** - Check logs for DDoS patterns (see `ddos_attack` category)
3. For databases: Look for "slow query" logs, "full table scan" warnings, or "missing index" errors
4. Correlate CPU spike with query latency increases
5. Service may still be responding but with severe performance degradation

**`memory_leak`** - Progressive memory consumption over time, trending toward Out of Memory (OOM) condition.

**`disk_full`** - Disk storage capacity exhausted, preventing writes and causing service failures.

**Important distinction - Disk Full vs Bad Configuration:**

- ❌ **WRONG**: Debug logging was enabled in production, filling the disk → `bad_configuration`
- ✅ **CORRECT**: Debug logging was enabled in production, filling the disk → `disk_full`
- ❌ **WRONG**: Log rotation misconfiguration caused disk to fill → `bad_configuration`
- ✅ **CORRECT**: Log rotation misconfiguration caused disk to fill → `disk_full`

**Key reasoning:**

- Diagnose based on the **current resource state**, not what caused it
- If the disk is full and that is what's preventing writes → `disk_full`
- `bad_configuration` applies when the misconfiguration is the **ongoing active problem** (wrong endpoint, wrong timeout, wrong feature flag causing current failures)
- When a configuration decision *in the past* caused a resource to be exhausted, diagnose the **resource exhaustion** (`disk_full`), not the configuration that triggered it
- Ask: "What is broken RIGHT NOW?" — a full disk is broken right now; the logging config was just what filled it

**`connection_pool_exhausted`** - Database connection pool depleted, preventing new database connections.

**Attribution Rule:** The `affected_service` must be the **application service** that is leaking or holding connections, NOT the database experiencing connection exhaustion. Connection pool exhaustion is caused by application code failing to properly release connections, creating "zombie" connections that never return to the pool.

**How to identify the correct service:**

1. Check database logs to identify which application user/service owns the excessive connections
2. Check that application service's logs for connection pool errors (e.g., "ConnectionAcquisitionException", "Pool is empty", "Timeout waiting for connection")
3. The application service exhibiting connection pool errors is the `affected_service`
4. The database is a **victim** showing symptoms, not the root cause

**Example scenarios:**

- ✅ Application service shows "Pool exhausted" errors + Database shows "too many connections from app_user" → `affected_service` = application service
- ❌ Database reports connection limit reached → Do NOT set `affected_service` = database
- ✅ Database indicates 100 connections owned by a specific application user → `affected_service` = that application service

**`thread_pool_exhausted`** - Application thread pool depleted, preventing request processing.

### Configuration Issues

**`bad_configuration`** - Incorrect service configuration (settings, parameters, feature flags) causing malfunction.

**Attribution Rule:** The `affected_service` is the service whose configuration is incorrect, not downstream services experiencing failures as a result.

**Common sources of bad configuration:**

1. **Feature flags** - Experimental or buggy features enabled via feature flag toggles
2. **Deployment configuration** - New deployment introduces bad config files or parameters
3. **Manual configuration changes** - Runtime config updates, parameter changes
4. **Environment mismatches** - Wrong settings for production vs staging

**Important distinction - Bad Configuration vs Data Corruption:**

- ❌ **WRONG**: Deployment introduces buggy code that writes corrupt data → `bad_configuration`
- ✅ **CORRECT**: Deployment introduces buggy code that writes corrupt data → `data_corruption`
- ❌ **WRONG**: Service writing negative values after deployment → `bad_configuration`
- ✅ **CORRECT**: Service writing negative values after deployment → `data_corruption`

**Key reasoning:**

- Focus on **WHAT the service is DOING**, not **HOW it got into that state**
- If the service is **actively writing invalid/corrupt data** → `data_corruption`
- If the service has **wrong settings/parameters** → `bad_configuration`
- Deployment timing does NOT determine the category

**Examples to clarify:**

- ✅ Wrong database connection string in config → `bad_configuration`
- ✅ Incorrect timeout value (30s instead of 300s) → `bad_configuration`
- ✅ Feature flag enabled that causes service to use wrong API endpoint → `bad_configuration`
- ❌ Bug in code that writes negative inventory values → NOT `bad_configuration`, use `data_corruption`
- ❌ Race condition creating duplicate orders → NOT `bad_configuration`, use `data_corruption`

**How to identify bad configuration:**

1. Check logs for "Configuration reloaded", "Feature flag [NAME] is now ON/OFF", or config-related messages
2. Look for error messages indicating missing fields, null values, or validation failures that suggest config issues
3. Check if the problem is with **settings/parameters** (configuration) vs **data being written** (corruption)
4. Ask: "Is the service misconfigured OR is it corrupting data?" - These are mutually exclusive
5. The service exhibiting errors due to its own configuration is the `affected_service`

**`missing_environment_variable`** - Required environment variable not set, preventing proper service initialization or operation.

**`invalid_credentials`** - Authentication credentials incorrect, expired, or revoked.

### Network & Connectivity

**`network_connectivity_issue`** - Network communication failure between services or external dependencies.

**Important distinction - Network Connectivity Issue vs DNS Resolution Failure:**

- ❌ **WRONG**: Logs show DNS failures + timeouts + TCP resets = `dns_resolution_failure`
- ✅ **CORRECT**: Multiple types of network failures together = `network_connectivity_issue`
- Use `network_connectivity_issue` when you see **multiple network problems** occurring together
- Use `dns_resolution_failure` only when DNS is the **sole** isolated problem

**How to identify network connectivity issue:**

1. **Multiple failure types** - Combination of DNS failures, connection timeouts, TCP resets, SSL handshake failures
2. **Retry loops** - Service stuck retrying failed connections, exhausting connection pools
3. **Circuit breaker OPEN** - Circuit breaker trips due to sustained network error rate
4. **Connection pool exhaustion** - All connections consumed by retry attempts
5. **Intermittent failures** - Network problems come and go, not consistently failing

**Common log patterns for network connectivity issue:**

- "Connection timeout to [external-api]"
- "DNS resolution intermittent for [domain]"
- "TCP connection reset by peer"
- "Failed to establish SSL handshake"
- "Connection pool exhausted - all connections in retry state"
- "Retry attempt X/50" (sustained retry loops)
- "Circuit breaker is OPEN due to network error rate"

**When to use `dns_resolution_failure` instead:**

- DNS is the **only** problem mentioned in logs (no timeouts, resets, or other failures)
- All errors are DNS-specific ("DNS lookup failed", "cannot resolve hostname")
- No evidence of connection attempts succeeding then failing (pure DNS issue)

**Network Connectivity Issue typically requires:**

- `service-restart` to break retry loops and reset connection state
- Not escalation (unless persistent after restart)

**`network_mesh_latency`** - Service mesh or inter-service communication experiencing high latency.

**`ddos_attack`** - Distributed Denial of Service attack or malicious traffic overwhelming the service.

**Important distinction - DDoS Attack vs CPU Saturation:**

- ❌ **WRONG**: High CPU + high requests from single IP = `cpu_saturation`
- ✅ **CORRECT**: High CPU + high requests from single IP = `ddos_attack`
- DDoS attacks **cause** CPU saturation, but the diagnosis should identify the **root cause** (attack), not the symptom (high CPU)

**How to identify DDoS attack:**

1. **Volumetric traffic spike** - Request rate dramatically higher than normal (e.g., 150 → 9200 requests)
2. **Single source pattern** - Multiple requests from same IP address or IP range
3. **Suspicious user agents** - Bot-like user agents (e.g., "BadBot/1.0", "Scanner/2.0")
4. **Rate limit warnings** - Logs show "Rate limit exceeded for IP X.X.X.X"
5. **Repeated identical requests** - Same endpoint hit repeatedly from same source
6. **Performance degradation** - CPU saturation, latency spikes, timeouts as secondary effects

**Log patterns that indicate DDoS:**

- Multiple access log entries from the same IP in short timeframe
- "Rate limit exceeded for IP X.X.X.X"
- Suspicious user agent strings in access logs
- "Worker process pinned to 100% CPU" combined with volumetric traffic

**Key reasoning:**

- If you see high CPU/latency/memory AND abnormal traffic patterns from specific IPs → `ddos_attack`
- The high resource usage is a **symptom** of the attack, not the primary diagnosis
- DDoS attacks require security intervention (IP blocking), not service actions (scaling/restart)

**`rate_limit_exceeded`** - Service hitting rate limits on external APIs or internal resource quotas.

**`dns_resolution_failure`** - DNS lookup failures preventing service discovery or external communication.

**Important:** Use this category **only** when DNS is the isolated problem. If logs show DNS failures **plus** connection timeouts, TCP resets, or other network errors, use `network_connectivity_issue` instead (see above).

### Cache & Data

**`stale_cache`** - Cache layer (Redis, Memcached, application cache) serving outdated data, causing incorrect application behavior.

**Common manifestations:**

1. **Missing or infinite TTL** - Cache entries with TTL=-1 or no expiration set
2. **Cache not invalidated** - Updates to source data don't trigger cache refresh
3. **Cache expiration too long** - TTL set too high for data volatility
4. **Manual cache updates missed** - Promotions, pricing changes not reflected in cache

**Important distinction - Stale Cache vs Bad Configuration:**

- ❌ **WRONG**: Cache with TTL=-1 is misconfigured → `bad_configuration`
- ✅ **CORRECT**: Cache with TTL=-1 serving stale data → `stale_cache`
- ❌ **WRONG**: Application code not setting TTL properly → `bad_configuration`
- ✅ **CORRECT**: Application code not setting TTL properly → `stale_cache`

**Key reasoning:**

- Focus on the **symptom** (stale data being served from cache)
- The root cause may be application code bug, infrastructure misconfiguration, or missing invalidation logic
- All cache-related staleness issues use this category
- The affected_service is typically the application service using the cache, not the cache infrastructure itself

**How to identify stale cache:**

1. **Check for cache hits with old data** - Logs show cache serving data that doesn't match source of truth (database)
2. **Look for TTL=-1 or no expiration** - Cache keys without proper expiration
3. **Database vs cache mismatch** - Database has correct data, cache has old data
4. **Timing correlation** - Issue started after price change, promotion end, or content update
5. **Cache invalidation failures** - Updates not triggering cache refresh

**Common log patterns:**

- "Cache hit for product_123: price=$50" (when database shows $100)
- "Redis key TTL: -1 (no expiration)"
- "Cache not invalidated after price update"
- "Stale pricing data in catalog cache"
- "Database shows correct value but cache serving old data"

**Examples to clarify:**

- ✅ Redis cache with TTL=-1 serving old prices → `stale_cache`
- ✅ Application cache not expiring after promotion ends → `stale_cache`
- ✅ Cache invalidation logic missing from code → `stale_cache`
- ✅ Memcached serving outdated product descriptions → `stale_cache`
- ✅ Black Friday prices still in cache after sale ended → `stale_cache`
- ❌ Database replication lag causing stale reads → NOT `stale_cache`, may need different category
- ❌ CDN serving old static assets after deployment → Consider deployment issue, not cache staleness

**`data_corruption`** - Service actively writing invalid, malformed, or corrupt data to datastores, compromising data integrity.

**Important distinction - Data Corruption vs Bad Configuration:**

- ❌ **WRONG**: Bug introduced by deployment that corrupts data → `bad_configuration`
- ✅ **CORRECT**: Bug introduced by deployment that corrupts data → `data_corruption`
- ❌ **WRONG**: Temporal correlation with deployment means it's configuration → `bad_configuration`
- ✅ **CORRECT**: Look at service behavior, not trigger timing → `data_corruption` if writing bad data

**Key reasoning:**

- Diagnosis category describes **WHAT IS HAPPENING**, not what caused it
- A deployment can introduce EITHER a bug (data corruption) OR wrong config (bad configuration)
- Ask: "What is the service actively DOING right now?"

**Common manifestations of data corruption:**

1. **Invalid values** - Writing negative numbers, null values where not allowed, out-of-range values
2. **Duplicate records** - Race conditions creating duplicate database entries
3. **Constraint violations** - Writing data that violates foreign key or unique constraints
4. **Malformed data** - Writing invalid JSON, corrupted binary data, truncated strings
5. **Data inconsistencies** - Writing data that breaks referential integrity across tables

**How to identify data corruption:**

1. **Check WHAT is being written** - Look for evidence of invalid data in logs
2. **Look for constraint violations** - Database rejecting writes due to integrity checks
3. **Check for duplicate creation** - Same entity being created multiple times
4. **Evidence in error logs** - "Negative inventory value", "Duplicate order ID", "Invalid data format"
5. **Focus on behavior, not timing** - Even if it started after a deployment, if the service is writing corrupt data, it's `data_corruption`

**Common error patterns for data corruption:**

- "ERROR: check constraint violated: inventory >= 0"
- "ERROR: duplicate key value violates unique constraint"
- "WARNING: Writing negative inventory value: -500 for product_991"
- "ERROR: Foreign key constraint failed"
- "Created duplicate order record: order_12345 already exists"

**Examples to clarify:**

- ✅ Service writing negative inventory values to database → `data_corruption`
- ✅ Race condition causing duplicate order records → `data_corruption`
- ✅ Service corrupting database after deployment of buggy code → `data_corruption`
- ✅ Bug in recommendation-api writing invalid data → `data_corruption`
- ❌ Service has wrong database connection string → NOT `data_corruption`, use `bad_configuration`
- ❌ Wrong timeout causing requests to fail → NOT `data_corruption`, use `bad_configuration`

**`schema_mismatch`** - Database schema doesn't match application expectations, causing query failures.

**What qualifies as schema mismatch:**

1. **Missing columns** - Application expects column that doesn't exist in database
2. **Wrong data types** - Column type changed but application code not updated
3. **Missing tables** - Application references tables that don't exist
4. **Constraint violations** - Foreign key constraints or unique constraints changed

**What is NOT schema mismatch:**

- ❌ **Missing indexes** - This is a performance issue (`cpu_saturation`), not schema incompatibility
- ❌ **Slow queries** - Performance problem, not schema problem
- ❌ **Missing data** - Data issue, not schema issue

**How to identify schema mismatch:**

1. Look for errors like: "column does not exist", "unknown column", "table not found"
2. Check for errors after deployment or migration
3. Errors reference specific schema elements (columns, tables, constraints)
4. Usually causes immediate query failures, not just slow performance

**Common error patterns:**

- `ERROR: column "user_email" does not exist`
- `ERROR: relation "users_v2" does not exist`
- `ERROR: invalid input syntax for type integer: "string_value"`

**`database_deadlock`** - Database transactions stuck in deadlock, causing timeouts and failures.

### Service State

**`service_crash`** - Service process has terminated unexpectedly and is not running.

**IMPORTANT - When to use `service_crash` vs specific root cause:**

A crash is a **symptom**, not always a diagnosis. Use `service_crash` **ONLY** when:

- Service is down/not running (CPU=0%, status=Error, connection refused) AND
- You **cannot determine WHY it crashed** from available logs, metrics, or recent changes
- It's a "mystery crash" with no clear evidence

**If you CAN identify the root cause, diagnose that instead:**

- ❌ **WRONG**: Logs show "OOMKilled" → `service_crash`
- ✅ **CORRECT**: Logs show "OOMKilled" + memory climbed to 99% → `memory_leak`
- ❌ **WRONG**: Crashed after deployment 5 min ago → `service_crash`
- ✅ **CORRECT**: Crashed after deployment + config error in logs → `bad_configuration`
- ❌ **WRONG**: Segfault with no logs or metrics → Could be `service_crash` OR `undiagnosable`
- ✅ **CORRECT**: Recent crash, metrics lag, no logs available → `service_crash` (known crash, unknown cause)

**Examples of valid `service_crash` usage:**

1. Service stopped responding 2 minutes ago, but metrics haven't updated (lag) and no recent logs
2. Process terminated with generic error, no OOM/config/deployment evidence
3. Kubernetes pod restart with no explanation in logs

**Examples where you should NOT use `service_crash`:**

1. OOMKilled by Kubernetes → Use `memory_leak` (cause is clear)
2. Crashed immediately after bad config deployment → Use `bad_configuration` (cause is clear)
3. SSL handshake failure causing service to fail → Use `expired_certificate` or `invalid_credentials` (cause is clear)

**`circuit_breaker_state`** - Circuit breaker stuck in non-optimal state (e.g., HALF_OPEN) preventing normal traffic flow.

### Security

**`expired_certificate`** - SSL/TLS certificate has expired, blocking secure connections.
