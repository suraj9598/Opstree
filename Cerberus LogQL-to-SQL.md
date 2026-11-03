# Cerberus LogQL-to-SQL POC with ClickHouse

> **Document purpose:** This README describes the log-query compatibility POC: Grafana sends Loki-compatible LogQL to Cerberus, Cerberus translates the query into ClickHouse SQL, and ClickHouse executes it against OpenTelemetry logs.
> 
> **Scope:** Logs only for this Cerberus POC. Metrics and traces are outside the migration scope of this POC.


## 1. Executive Summary

The goal is to demonstrate that a Grafana query written in LogQL can be accepted by a Loki-compatible Cerberus endpoint, translated into SQL, and executed against ClickHouse.

The intended compatibility path is:

1. Grafana continues using a **Loki datasource**.
2. The datasource endpoint is pointed/routed to **Cerberus Loki API** rather than a Loki backend.
3. Grafana sends LogQL through Loki-compatible API routes, such as `/loki/api/v1/query_range`.
4. Cerberus parses the LogQL expression and generates ClickHouse SQL.
5. Cerberus connects to ClickHouse using the native protocol (`:9000`) in the POC.
6. ClickHouse reads from `observability.otel_logs` and returns records/series.
7. Cerberus responds using Loki-compatible response format, which Grafana renders.

**Demo success criterion:** Show the same LogQL expression in Grafana and in Cerberus DEBUG logs, show the generated SQL beside it in Cerberus logs, and then find the executed SQL in ClickHouse `system.query_log`.

This POC is a query-compatibility proof, not a production-readiness or performance benchmark.

## 2. Problem Statement

The existing observability environment has dashboards that query Loki using LogQL. Recreating dashboards and rewriting all panels as ClickHouse SQL would increase migration effort and create dashboard maintenance overhead.

The POC investigates whether a compatibility gateway can preserve the Grafana/Loki query interface while ClickHouse becomes the log-query backend.

### Desired outcome

- Keep Grafana and its existing dashboards.
- Keep the Grafana data source type as **Loki**.
- Introduce Cerberus as the Loki-compatible query endpoint.
- Store/query logs in ClickHouse (`observability.otel_logs`).

### What this POC does not claim

- It does not prove that all LogQL features are translated correctly.
- It does not prove that every existing dashboard/panel will work without modification.
- It does not migrate Tempo or VictoriaMetrics.
- It does not establish production scalability, HA, cost savings, or a definitive latency advantage.
- A successful basic query is proof for that query path, not full LogQL parity.

## 3. Architecture

### 3.1 Logical architecture

```text
                 Existing Grafana (external VM)
                 ┌────────────────────────────┐
                 │ Existing dashboards        │
                 │ Loki datasource            │
                 │ LogQL expression           │
                 └──────────────┬─────────────┘
                                │ Loki-compatible HTTP API
                                │ query / query_range / labels
                                ▼
                 ┌────────────────────────────┐
                 │ Cerberus Loki API head    │
                 │ Parse LogQL               │
                 │ Build ClickHouse SQL       │
                 │ DEBUG logs: LogQL + SQL   │
                 └──────────────┬─────────────┘
                                │ ClickHouse native protocol
                                │ TCP :9000
                                ▼
                 ┌────────────────────────────┐
                 │ ClickHouse                │
                 │ Database: observability    │
                 │ Table: otel_logs           │
                 │ system.query_log           │
                 └────────────────────────────┘
```

### 3.2 Request flow explained

**Step A — Grafana sends the query**

Grafana’s Loki datasource serializes the selected query and time range into a Loki API request. Grafana does not need to understand ClickHouse SQL for this path.

**Step B — Cerberus receives a Loki-compatible request**

The Cerberus Loki endpoint receives API requests such as `query_range`. Its DEBUG output includes fields such as:
- `msg`: request class, for example `cerberus loki query_range`
- `api`: `loki`
- `logql`: original query expression
- `sql`: generated SQL statement
- `args`: bound values used with SQL placeholders

**Step C — Cerberus translates LogQL to SQL**

For a selector such as:

```logql
{k8s_namespace_name="clickhouse"}
```

Cerberus builds a ClickHouse SELECT that reads log body and attributes from `otel_logs`, applies namespace and time filtering, and returns the result shape expected by the Loki API.

The generated SQL can be long because it includes attribute/label mapping and query-range semantics. The query is parameterized; `?` placeholders and the accompanying `args` field are part of the debug evidence.

**Step D — ClickHouse executes the SQL**

Cerberus uses the ClickHouse Go client. The ClickHouse query history shows `client_name` similar to:

```text
clickhouse-go/2.49.0 (lv:go/1.26.2; os:linux)
```

The `system.query_log` record proves SQL execution, while Cerberus DEBUG log provides the original LogQL-to-SQL relationship.

**Step E — Cerberus returns Loki-shaped results**

Grafana consumes the Loki-compatible response and renders logs or time series in Explore/dashboard panels.

## 4. Components and Responsibilities

| Component | Role in this POC |
|---|---|
| Grafana (external VM) | UI, Explore, dashboards, and Loki datasource client |
| Loki datasource configuration | Preserves the existing datasource interface; endpoint must resolve to Cerberus |
| Cerberus | Loki-compatible API and LogQL-to-ClickHouse SQL translation |
| ClickHouse | SQL execution and storage/query backend for logs |
| `observability.otel_logs` | OpenTelemetry log table queried by Cerberus |
| `system.query_log` | ClickHouse-side SQL execution evidence |
| Kubernetes / Helm | Runs ClickHouse and Cerberus workloads in the isolated test environment |

**Important:** Grafana remains outside the Kubernetes test environment. Therefore network routing/forwarding from the Grafana VM to the Cerberus service is an essential part of the integration.

## 5. Environment and Configuration Notes

### 5.1 Namespaces

Namespaces observed for the focused setup include:

- `clickhouse` — ClickHouse deployment and related resources
- `cerberus` — Cerberus deployment/service
- `logging` — telemetry/logging components that were present in the test setup

Confirm the live namespace/resource state before the demo:

```bash
kubectl get namespaces
kubectl get pods -n clickhouse
kubectl get pods -n cerberus
kubectl get svc -n cerberus
```

### 5.2 ClickHouse connection

Cerberus connection values used/targeted in the project discussion:

| Setting | Value / meaning |
|---|---|
| ClickHouse address | `clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000` |
| Protocol | `native` |
| Database | `observability` |
| Log table | `otel_logs` |
| Username | `default` in the referenced configuration |
| Port | `9000` native protocol; HTTP interface is typically `8123` |

The service DNS name is Kubernetes-internal. Cerberus is inside the cluster, so it can use this service address directly. Grafana does **not** connect directly to this ClickHouse native address for the Loki query path.

### 5.3 Cerberus Helm chart

The chart artifact examined in the project was:

```text
oci://ghcr.io/tsouza/cerberus/charts/cerberus
```

Version examined:

```text
0.19.1
```

Chart values were reviewed using commands like:

```bash
helm show readme oci://ghcr.io/tsouza/cerberus/charts/cerberus --version 0.19.1
helm show values oci://ghcr.io/tsouza/cerberus/charts/cerberus --version 0.19.1
```

The values file discussed in the terminal was named:

```text
~/oct/cerberus-default-values.yaml
```

The chart supports split API heads (including a Loki head) and shared ClickHouse connection configuration. The deployed workload observed in logs was:

```text
deployment/cerberus-loki
namespace: cerberus
```

The exact complete final values file should be included in the repository alongside this README if the POC is to be reproduced byte-for-byte. This README intentionally does not invent unobserved values.

### 5.4 Debug logging

For the demo, Cerberus DEBUG logs are the key translation evidence. Use:

```bash
kubectl logs -f deployment/cerberus-loki -n cerberus --since=2m
```

Or filter for relevant request records:

```bash
kubectl logs deployment/cerberus-loki -n cerberus --since=5m \
  | grep -E 'query_range|logql|labels'
```

Do not leave verbose DEBUG logging enabled indefinitely in a production environment. It can increase log volume and may expose query details/attributes.

## 6. Setup Walkthrough (Reproducibility Runbook)

> The commands below are a structured runbook. Confirm the repository’s actual manifests, Helm values, and cluster-specific configuration before re-running. Any `<...>` item is a placeholder and must be replaced with the actual local value.

### 6.1 Confirm Kubernetes context

```bash
kubectl config current-context
kubectl get nodes
```

Purpose:
- Ensure commands are applied to the intended isolated POC context.
- Confirm the Kubernetes node is Ready before installing workloads.

### 6.2 Create/confirm required namespaces

```bash
kubectl create namespace clickhouse --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace cerberus --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace logging --dry-run=client -o yaml | kubectl apply -f -
```

Purpose: ensure namespaces exist without failing if they already exist.

### 6.3 Install/confirm ClickHouse

The POC environment uses ClickHouse managed in Kubernetes, with the ClickHouse Operator and Keeper discussed as part of the database platform setup. Use the project’s retained operator/keeper and ClickHouse custom-resource manifests for the exact installation. Then validate:

```bash
kubectl get pods -n clickhouse
kubectl get svc -n clickhouse
```

Connect to the ClickHouse client pod:

```bash
kubectl exec -it -n clickhouse \
  clickhouse-clickhouse-0-0-0 -- clickhouse-client
```

Basic SQL health check:

```sql
SELECT 1;
```

Check database and log table:

```sql
SHOW DATABASES;
SHOW TABLES FROM observability;
DESCRIBE TABLE observability.otel_logs;
```

Confirm that the table has data:

```sql
SELECT
    count() AS total_rows,
    max(Timestamp) AS latest_log
FROM observability.otel_logs;
```

A nonzero row count and recent `latest_log` indicate that logs are present. A table existing alone does not prove ingestion is current.

### 6.4 Confirm Cerberus configuration and install/upgrade

Before deployment, inspect the relevant values and rendered Kubernetes resources:

```bash
helm show values oci://ghcr.io/tsouza/cerberus/charts/cerberus \
  --version 0.19.1 > /tmp/cerberus-chart-values.yaml

helm template cerberus-loki \
  oci://ghcr.io/tsouza/cerberus/charts/cerberus \
  --version 0.19.1 \
  -n cerberus \
  -f ~/oct/cerberus-default-values.yaml
```

Purpose:
- First command saves chart defaults for reference.
- Second command renders YAML locally for review without applying it.
- Check ClickHouse hostname/database/protocol, Loki API head, service ports, and debug level in the rendered output.

Deploy or update only after reviewing the values:

```bash
helm upgrade --install cerberus-loki \
  oci://ghcr.io/tsouza/cerberus/charts/cerberus \
  --version 0.19.1 \
  -n cerberus \
  -f ~/oct/cerberus-default-values.yaml
```

Confirm rollout:

```bash
kubectl rollout status deployment/cerberus-loki -n cerberus
kubectl get pods,svc -n cerberus
kubectl logs deployment/cerberus-loki -n cerberus --since=5m
```

**Note:** The exact release name and values file must match the actual environment. The deployment name above is the observed name from the logs.

### 6.5 Network path from Grafana VM

Grafana runs on a separate VM. It must be able to reach Cerberus’s Loki-compatible HTTP service.

For local testing, a port-forward can be used from a machine that has Kubernetes access:

```bash
kubectl port-forward -n cerberus svc/<cerberus-loki-service> 3100:8080
```

- `3100` is the local listening port selected for the test.
- `8080` is the Cerberus service target port in the chart’s standard service configuration; verify the actual service before using.
- This port-forward only serves the host where it runs. For a remote Grafana VM, the endpoint needs an appropriate route, SSH tunnel, reverse tunnel, or reachable service exposure.

Do not set the Grafana datasource to a `127.0.0.1` address unless the port-forward/tunnel is running in the same network namespace as Grafana itself. `localhost` on the Grafana VM is not the developer laptop’s localhost.

### 6.6 Grafana Loki datasource configuration

In Grafana:
1. Open **Connections / Data sources**.
2. Select the existing Loki datasource (the datasource type remains Loki).
3. Configure its URL to the reachable Cerberus HTTP endpoint, not the ClickHouse native endpoint.
4. Save and test.
5. Open **Explore**, select the Loki datasource, and run a simple selector.

Use this initial query:

```logql
{k8s_namespace_name="clickhouse"}
```

This query selects logs by the Kubernetes namespace attribute. It is a better initial smoke test than adding a text filter such as `|= "error"` before confirming the base selector returns records.

The exact datasource URL depends on the forwarding/routing used in the environment; keep it as an environment-specific value rather than hardcoding an unverified address in this README.

In Grafana Explore:
- Choose the existing Loki data source.
- Enter:

```logql
{k8s_namespace_name="clickhouse"}
```

- Set a practical time range (for example, the last 5 minutes).
- Run the query

Narration:
> “Grafana is still speaking the Loki API and the query is still LogQL. We have not rewritten the dashboard into ClickHouse SQL.”

Run in terminal:

```bash
kubectl logs deployment/cerberus-loki -n cerberus --since=2m \
  | grep -E 'query_range|logql|labels'
```

Look for a structured DEBUG record with:

```text
msg: cerberus loki query_range
api: loki
logql: {k8s_namespace_name="clickhouse"}
sql: SELECT ... FROM `otel_logs` ...
args: [...]
```

Explanation:
- `logql` is the original expression received by Cerberus.
- `sql` is the SQL Cerberus generated.
- `args` carries values bound to placeholders.
- Some additional `cerberus loki labels` events are metadata/label-discovery queries; those are expected and are not the log-fetch query itself.

Open ClickHouse client:

```bash
kubectl exec -it -n clickhouse \
  clickhouse-clickhouse-0-0-0 -- clickhouse-client
```

Run the query below shortly after clicking Run in Grafana:

```sql
SELECT
    event_time,
    query_id,
    query_duration_ms,
    exception,
    query
FROM system.query_log
WHERE event_time >= now() - INTERVAL 5 MINUTE
  AND type = 'QueryFinish'
  AND client_name LIKE '%clickhouse-go%'
  AND query LIKE '%otel_logs%'
  AND query NOT LIKE 'INSERT%'
ORDER BY event_time DESC
LIMIT 10;
```

Interpretation:
- `client_name LIKE '%clickhouse-go%'` narrows the records to the Go client Cerberus uses.
- `query LIKE '%otel_logs%'` selects statements touching the log table.
- `type = 'QueryFinish'` selects completed queries.
- Empty `exception` means the recorded query completed without a reported exception.
- `query_duration_ms` is the ClickHouse server-side query duration.
- This query may include label-discovery SQL as well as log fetches. The `arrayJoin(ResourceAttributes.keys)` query is a label discovery query; identify the main SELECT separately.

**Evidence limitation:** ClickHouse `system.query_log` records the SQL that ran, but it does not necessarily contain the original LogQL text. Use Cerberus DEBUG logs to show LogQL and generated SQL in the same record, and use ClickHouse query history to demonstrate that SQL execution occurred.

### Note:

Use the Cerberus timestamp and the ClickHouse `event_time` to identify a close-in-time SQL event. Where useful, compare query IDs and query text. Do not claim an exact correlation solely because multiple queries reference `otel_logs`; Grafana may trigger labels, metadata, and range-query requests during one interaction.

## 7. Validation Commands and SQL

### 7.1 Kubernetes status

```bash
kubectl get pods -n clickhouse
kubectl get pods -n cerberus
kubectl get svc -n cerberus
kubectl describe deployment cerberus-loki -n cerberus
```

### 7.2 ClickHouse connection

```sql
SELECT 1;
```

### 7.3 Log ingestion freshness

```sql
SELECT
    count() AS total_rows,
    max(Timestamp) AS latest_log
FROM observability.otel_logs;
```

### 7.4 Check ClickHouse query history for Cerberus client

```sql
SELECT
    event_time,
    type,
    query_id,
    client_name,
    query_duration_ms,
    read_rows,
    read_bytes,
    exception,
    query
FROM system.query_log
WHERE event_time >= now() - INTERVAL 10 MINUTE
  AND type = 'QueryFinish'
  AND client_name LIKE '%clickhouse-go%'
  AND query LIKE '%otel_logs%'
ORDER BY event_time DESC
LIMIT 20;
```

This broader query is useful during troubleshooting. It may show more than one SQL category, including metadata discovery and aggregation queries.

### 7.5 Distinguish label discovery from log fetch

A label discovery statement may look like:

```sql
SELECT DISTINCT arrayJoin(`ResourceAttributes`.`keys`) AS `k`
FROM `otel_logs`
WHERE `Timestamp` >= ...
  AND `Timestamp` <= ...
ORDER BY `k`;
```

This query enumerates resource-attribute keys. It is not the same as a query returning log bodies.

A log fetch statement often has a shape resembling:

```sql
SELECT `Body` AS `Line`, ...
FROM `otel_logs`
PREWHERE (`__otel_materialized_k8s.namespace.name` = ?)
WHERE (`Timestamp` >= ...)
ORDER BY `Timestamp` DESC
LIMIT ...;
```

The exact SQL varies with LogQL expression, selected labels, time range, and Cerberus translation implementation.

## 8. Observed Evidence from the POC

### 8.1 Cerberus DEBUG logs (observed)

A query-range event was observed with:

```logql
{k8s_namespace_name="clickhouse"}
```

and a generated SQL statement reading from:

```text
otel_logs
```

The SQL included a namespace filter using the materialized Kubernetes namespace attribute:

```text
__otel_materialized_k8s.namespace.name = ?
```

with an argument corresponding to:

```text
clickhouse
```

A separate event for a LogQL aggregation was observed:

```logql
sum by (level, detected_level) (
  count_over_time({k8s_namespace_name="clickhouse"} | drop __error__[1s])
)
```

Cerberus generated SQL with aggregate/bucket logic and attribute grouping. This is evidence that the POC exercised both a basic log selector and a LogQL aggregation pattern; it is not proof of full LogQL feature coverage.

### 8.2 ClickHouse `system.query_log` (observed)

A query result captured on 2026-11-03 showed:

| Field | Observed value |
|---|---|
| `event_time` | `2026-11-03 11:34:12` |
| `query_id` | `f93d8d12dc4797e4899a7e16d230906c-ea4e9fddd1f8d3c5-79` |
| `query_duration_ms` | `6` |
| `exception` | empty |
| query type | `SELECT DISTINCT arrayJoin(ResourceAttributes.keys) ... FROM otel_logs` |


### 8.3 What this evidence establishes

- Cerberus received Loki API requests.
- Cerberus DEBUG logging captured LogQL and generated SQL in a query-range event.
- Generated SQL targets ClickHouse `otel_logs`.
- ClickHouse query history contains completed SQL executed through the `clickhouse-go` client.
- At least the tested query types reached the backend successfully without an exception in the recorded event.

### 8.4 What still requires additional validation

- Capture a clean end-to-end log-fetch query in `system.query_log` immediately after Grafana Explore runs the selector.
- Validate representative dashboard queries beyond a simple selector and a tested aggregation.
- Test filters, regexes, parsing stages, label transformations, range aggregation semantics, and edge cases expected in the existing dashboards.
- Verify result correctness against Loki for a fixed time interval and stable sample set.
- Evaluate operational behavior under realistic concurrent query/load volume.

## 9. Troubleshooting Guide

| Symptom | Likely checks |
|---|---|
| Grafana “No logs found” | Start with the bare selector `{k8s_namespace_name="clickhouse"}`; confirm the attribute name exists and the time range is recent |
| Grafana datasource Save & Test fails | Verify URL is reachable from the Grafana VM, and that the endpoint is Cerberus HTTP service—not ClickHouse native port 9000 |
| Cerberus pod not Ready | Inspect `kubectl describe pod`, deployment logs, ClickHouse address, DB/table availability, and startup requirements checks |
| Cerberus logs show labels calls but no query_range | Confirm the Explore query was actually run and the selected datasource is the Loki datasource routed to Cerberus |
| ClickHouse query log shows only `arrayJoin(ResourceAttributes.keys)` | That is label discovery; trigger a log query and inspect newer rows, excluding metadata SQL if needed |
| ClickHouse query has an exception | Read the full `exception` field and compare Cerberus generated SQL and `args`; validate the table schema |
| Grafana VM cannot reach a local port-forward | Port-forward is local to the host where it runs; establish an appropriate SSH/network route from the Grafana VM |
| Query output empty after adding `|= "error"` | First validate base selector; then confirm the selected logs contain that literal text and the filter semantics are supported |

Useful commands:

```bash
kubectl get pods -n cerberus
kubectl describe deployment cerberus-loki -n cerberus
kubectl logs deployment/cerberus-loki -n cerberus --since=10m
kubectl get svc -n cerberus -o wide
kubectl get pods -n clickhouse
```

## 10. Limitations, Risks, and Production Considerations

### 10.1 LogQL compatibility is query-specific

Loki API compatibility does not automatically imply complete LogQL language parity. A production migration needs a query inventory from the dashboards and test coverage for every relevant operator/aggregation.

### 10.2 SQL translation and data model mapping

Loki labels and ClickHouse resource/log attributes do not necessarily have identical representation. Attribute key normalization, absent attributes, type mismatches, severity mapping, and high-cardinality labels need explicit validation.

### 10.3 Query performance and resource use

The 6 ms observation is one metadata SQL sample in a small POC environment. It must not be generalized as end-to-end Grafana latency or a production performance result. For fair testing, capture:
- fixed query set and time window
- repeated samples
- end-to-end Grafana response time
- ClickHouse query duration/read rows/read bytes/memory
- concurrency and data volume
- equivalent filters and output scope

### 10.4 Security

- Use authentication and least-privilege ClickHouse credentials.
- Avoid exposing Cerberus service publicly without TLS/auth/network policy.
- Treat debug logs as potentially sensitive because they can include query text, labels, and bound values.
- Store credentials in Kubernetes Secrets/secret references, not in README or checked-in values files.

### 10.5 Reliability and high availability

This POC validates a path in a local Kubernetes environment. It does not demonstrate production HA, multi-zone resilience, failover, backup/restore, capacity planning, or an SLO.

### 10.6 Existing dashboards

The aim is to preserve dashboards and datasource type, but panel-by-panel compatibility still needs verification. Some panels may rely on Loki-specific features or response shapes that Cerberus does not yet support or that need query adjustments.


### One-minute overview

> “We have an existing Grafana estate with Loki datasources and LogQL dashboards. Instead of rewriting those dashboards into SQL, this POC puts Cerberus behind the Loki-compatible API. Grafana sends LogQL; Cerberus translates it into ClickHouse SQL; ClickHouse executes against the OpenTelemetry log table; and Cerberus returns a Loki-shaped response. The key validation is that we can inspect the original LogQL and generated SQL in Cerberus debug logs, and verify the executed SQL in ClickHouse query history.”

### Why Cerberus

- Provides a Loki-compatible query interface to the POC.
- Acts as a translation layer between LogQL and ClickHouse SQL.
- Enables evaluation of ClickHouse as a backend without making Grafana dashboard rebuilding the first step.
- Makes the translation inspectable through structured debug logs.

### Why ClickHouse

- Central SQL engine for log analytics in this evaluation.
- Logs are queryable from a structured OpenTelemetry schema.
- `system.query_log` gives a backend-side view of SQL execution.

### Current conclusion

> “The POC demonstrates the basic compatibility flow for the queries tested. It is a promising migration path for logs, but broader LogQL compatibility, result parity, load testing, and production reliability remain validation items before recommending rollout.”


## 14. Commands Quick Reference

```bash
# Confirm cluster/context
kubectl config current-context
kubectl get nodes

# Resource status
kubectl get pods -n clickhouse
kubectl get pods -n cerberus
kubectl get svc -n cerberus

# Cerberus request/translation logs
kubectl logs -f deployment/cerberus-loki -n cerberus --since=2m
kubectl logs deployment/cerberus-loki -n cerberus --since=5m \
  | grep -E 'query_range|logql|labels'

# ClickHouse client
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 \
  -- clickhouse-client
```

```sql
-- Basic backend health
SELECT 1;

-- Log table freshness
SELECT count() AS total_rows, max(Timestamp) AS latest_log
FROM observability.otel_logs;

-- Cerberus-originated query execution records
SELECT
    event_time,
    query_id,
    query_duration_ms,
    exception,
    query
FROM system.query_log
WHERE event_time >= now() - INTERVAL 5 MINUTE
  AND type = 'QueryFinish'
  AND client_name LIKE '%clickhouse-go%'
  AND query LIKE '%otel_logs%'
  AND query NOT LIKE 'INSERT%'
ORDER BY event_time DESC
LIMIT 10;
```

---

## Appendix A — Terminology

| Term | Meaning |
|---|---|
| LogQL | Loki query language used by Grafana’s Loki datasource |
| SQL | Query language executed by ClickHouse |
| Cerberus | Loki-compatible API/translation gateway in this POC |
| `query_range` | Loki API route used to fetch log data over a time interval |
| `otel_logs` | OpenTelemetry logs table in the `observability` database |
| `system.query_log` | ClickHouse system table recording query execution metadata |
| Label discovery | Request flow to discover available labels/attribute keys; distinct from log retrieval |
| Native protocol | ClickHouse client protocol, configured at TCP port 9000 in this POC |

## Appendix B — Artifacts to Keep with 

- Cerberus Helm values file with passwords removed or represented through Secret references
- ClickHouse custom-resource/operator/keeper manifests used for the POC
- Grafana datasource configuration (endpoint anonymized as appropriate)
- Representative Cerberus DEBUG log excerpt showing LogQL and generated SQL
- ClickHouse `system.query_log` sample for the same demo run
