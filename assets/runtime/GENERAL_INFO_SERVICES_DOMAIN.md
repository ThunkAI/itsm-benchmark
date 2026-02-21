## Services Model

The list of services and their dependencies are accessible via specific service information tools.

## Data Freshness and Observability Limitations

When analyzing incidents, be aware of data collection lag and tool availability:

1. **Metric Collection Lag**: Metrics are typically collected every 5 minutes. Recent issues (< 5 minutes) may not appear in metric data yet.

2. **Log Ingestion Lag**: Logs may have ingestion delays. Check timestamps - gaps between the most recent log entry and the reported issue time suggest incomplete visibility.

3. **Verification Before Action**: Before taking remedial action (restart, scale, etc.), verify the current state:

   - If recent claims (< 5 min) cannot be verified with available data
   - AND you lack real-time verification tools (e.g., `service-check-health`)
   - THEN escalate rather than act on unverified information

4. **Acknowledge Limitations**: When escalating due to data lag or missing tools, explicitly state:
   - What you cannot verify (e.g., "Metrics show healthy but have 5-min lag")
   - Which tool would enable verification (e.g., "Lack service-check-health for real-time probe")
   - Why escalation is appropriate (e.g., "Cannot confirm current state before restart")

Example: A ticket claims a service crashed 2 minutes ago, but metrics show it healthy. The most recent metric is from 5 minutes ago. You should escalate, noting the lag prevents verification of current state.

Observability tools are available that provide information about:

- which services are present in the overall application
- the state of those services
- metrics that capture performance of the services
- logs that provide an account of the processes implemented by those services

There are additional observation tools that you don't have access to: {service-get-traces, service-check-health, service-get-active-alerts, service-get-certificate-info, service-get-env-vars}.

Note that if an issue isn't visible in a service’s metrics, it may be visible in the logs.

However, if there is no report of a problem with respect to some specific issue in the logs it likely means that that service does not have a problem of that nature.
For example: there are no metrics for disk usage and a system that has no disk capacity issues may not report any positive messages about disk health.

## Service Governance

There is a set of service control actions made available via tools.

There are additional service control actions that you don’t have access to and will have to escalate to a human: {service-set-power-state, service-deploy-version, service-flush-cache, service-set-feature-flag, service-reset-circuit-breaker, service-update-env-var, service-renew-certificate, service-block-ip-address}.

If the correct action to resolve an issue is to scale a service, you must exactly double the number of instances. You can use the service-get-info tool call to get the current provisioned instance count for a service.

If you observe db connection pool exhaustion or memory issues, the correct action to take is to restart the service.
