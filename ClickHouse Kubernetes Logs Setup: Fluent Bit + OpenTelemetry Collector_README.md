# ClickHouse Kubernetes Logs Setup: Fluent Bit + OpenTelemetry Collector

## 1. Overview

This README documents the steps performed in the ClickHouse observability POC to send Kubernetes container logs to ClickHouse through two separate collectors:

- **Fluent Bit (existing):** writes logs to `observability.kubernetes_logs`.
- **OpenTelemetry Collector (new DaemonSet):** reads Kubernetes pod log files and is configured to write logs to `observability.otel_logs`.

The purpose is to keep the existing Fluent Bit pipeline running while validating a separate OTel-based ingestion path. The existing OTel Collector deployment for metrics and traces was left unchanged.

> **Verification status:** The new `otel-logs` Helm release was installed, two agent pods were observed running, and `EXISTS TABLE observability.otel_logs` returned `1`. The OTel table's row count and sample records still need to be verified before declaring log ingestion fully successful.

## 2. Environment

| Component | Value |
|---|---|
| Kubernetes distribution | Kind |
| Kind cluster | `kind-clickhouse-poc` |
| Kubernetes context observed | `kind-kind-clickhouse-poc` |
| ClickHouse namespace | `clickhouse` |
| OTel Collector namespace | `monitoring` |
| Fluent Bit namespace | `logging` |
| ClickHouse pod | `clickhouse-clickhouse-0-0-0` |
| ClickHouse database | `observability` |
| ClickHouse HTTP service | `clickhouse-http`, NodePort `30081`, service port `8123` |
| ClickHouse native service | `clickhouse-clickhouse-headless`, port `9000` |
| Existing OTel Collector release | `otel-collector` |
| New OTel logs release | `otel-logs` |
| OTel Collector image | `otel/opentelemetry-collector-contrib:0.160.0` |
| OpenTelemetry Helm chart version | `0.173.1` |

These values reflect the environment used during this setup. Verify them before applying the instructions to another cluster.

## 3. High-level architecture and data flow

```text
Kubernetes container log files
        |
        +--> Fluent Bit (existing DaemonSet, namespace logging)
        |        |
        |        +--> ClickHouse HTTP service :8123
        |                 |
        |                 +--> observability.kubernetes_logs
        |
        +--> OTel Collector agents (new DaemonSet, namespace monitoring)
                 |
                 +--> ClickHouse native TCP service :9000
                          |
                          +--> observability.otel_logs
```

Both collectors can read the same Kubernetes log files, but they write to separate tables. This keeps their data separate for validation and comparison.

## 4. Existing Fluent Bit setup

Fluent Bit was already installed in namespace `logging` using Helm:

- Helm release: `fluent-bit`
- Chart version: `0.58.2`
- App version: `5.1.2`
- Pods observed: `fluent-bit-hrtjt` and `fluent-bit-vbhpn`

The configuration reads container log files from `/var/log/containers/*.log`, applies Kubernetes metadata filtering, and sends records to ClickHouse using an HTTP `INSERT ... FORMAT JSONEachRow` query.

Existing destination table:

```sql
observability.kubernetes_logs
```

Observed table columns:

```sql
timestamp DateTime,
namespace String,
pod String,
container String,
node String,
message String
```

**Important:** Fluent Bit was not disabled or reconfigured during the OTel logs setup.

## 5. Existing OTel Collector kept separate

The existing Helm release was `otel-collector` in namespace `monitoring`:

- Chart version: `0.173.1`
- Collector image: `otel/opentelemetry-collector-contrib:0.160.0`
- Mode: `deployment`

Its observed pipelines were:

- **Logs:** OTLP receiver → memory limiter/batch processors → `debug` exporter
- **Metrics:** Prometheus receiver → batch processor → ClickHouse exporter
- **Traces:** OTLP receiver → batch processor → ClickHouse exporter

This existing release was not changed. A separate release named `otel-logs` was installed to add file-based Kubernetes log collection as a DaemonSet.

## 6. Create the OTel logs values file

The file created was `~/otel-logs-values.yaml` with the following content:

```yaml
mode: daemonset

image:
  repository: otel/opentelemetry-collector-contrib
  tag: "0.160.0"

presets:
  logsCollection:
    enabled: true
    includeCollectorLogs: false
    storeCheckpoints: false

config:
  exporters:
    clickhouse:
      endpoint: tcp://clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000
      database: observability
      username: default
      async_insert: true
      create_schema: true
      logs_table_name: otel_logs

  service:
    pipelines:
      logs:
        receivers:
          - filelog
        processors:
          - memory_limiter
          - batch
        exporters:
          - clickhouse
```

### Key configuration settings

| Setting | Purpose |
|---|---|
| `mode: daemonset` | Runs an OTel agent on each eligible Kubernetes node. |
| `presets.logsCollection.enabled` | Enables the chart's Kubernetes log collection preset and required file mounts/configuration. |
| `includeCollectorLogs: false` | Avoids collecting the OTel Collector's own logs through this preset. |
| `storeCheckpoints: false` | Disables checkpoint persistence for this setup; reconsider for production restart/resume requirements. |
| `endpoint` | Uses ClickHouse's native TCP endpoint on port `9000`. |
| `database: observability` | Selects the ClickHouse database. |
| `logs_table_name: otel_logs` | Configures the ClickHouse exporter to use `observability.otel_logs` for logs. |
| `async_insert: true` | Enables asynchronous inserts in the exporter. |
| `create_schema: true` | Allows the exporter to create its required schema/table where supported. |
| `filelog` receiver | Reads Kubernetes container log files from mounted pod log paths. |
| `memory_limiter`, `batch` | Provide memory protection and batch records before export. |

The chart rendered the receiver as `file_log` for Collector version `0.160.0`. Helm warned that the old receiver name `filelog` was automatically rewritten to `file_log` and that the automatic rewrite may be removed in a future release. Review the rendered configuration and receiver naming when upgrading the chart or Collector version.

## 7. Validate the Helm rendering

Before installation, the chart was rendered to a local file:

```bash
helm template otel-logs open-telemetry/opentelemetry-collector \
  --version 0.173.1 \
  -n monitoring \
  -f ~/otel-logs-values.yaml \
  > /tmp/otel-logs-rendered.yaml
```

The rendering completed successfully. The output was inspected to confirm:

- the file receiver was configured (rendered as `file_log`);
- the logs pipeline used the file receiver and ClickHouse exporter;
- the endpoint was `clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000`;
- the database was `observability` and logs table was `otel_logs`;
- the DaemonSet mounted Kubernetes pod logs, including `/var/log/pods`.

Rendering before installation helps catch configuration problems before changing the cluster.

## 8. Install the separate OTel logs release

The release was installed with:

```bash
helm upgrade --install otel-logs open-telemetry/opentelemetry-collector \
  --version 0.173.1 \
  -n monitoring \
  -f ~/otel-logs-values.yaml \
  --wait \
  --timeout 3m
```

The command returned `STATUS: deployed` and `DESCRIPTION: Install complete` for release `otel-logs`, revision `1`, namespace `monitoring`.

Helm also warned that resource requests/limits were not configured and that `GOMEMLIMIT` was not set because no memory limit was defined. These were warnings, not installation failures. For production, define appropriate resource requests/limits based on expected log volume and measured resource usage.

## 9. Verify OTel agent pods

Command used:

```bash
kubectl get pods -n monitoring -o wide | grep otel-logs
```

The following two pods were observed in `Running` state:

```text
otel-logs-opentelemetry-collector-agent-7m2f2  1/1 Running  ... kind-clickhouse-poc-worker2
otel-logs-opentelemetry-collector-agent-gzx4w  1/1 Running  ... kind-clickhouse-poc-worker
```

This confirmed that the DaemonSet had an agent running on both worker nodes in this Kind cluster.

## 10. Inspect OTel Collector logs

Command used:

```bash
kubectl logs -n monitoring otel-logs-opentelemetry-collector-agent-gzx4w --since=5m
```

The output confirmed that Collector version `0.160.0` started, the `file_log` receiver watched files under `/var/log/pods/...`, and the Collector reported that it was ready to process data.

The inspected output did not show an obvious ClickHouse export error in the visible portion. Startup messages do not prove that log rows were inserted, so the ClickHouse table and contents must also be checked.

## 11. Verify ClickHouse table and data

### 11.1 Open a shell inside the ClickHouse pod

From the host terminal:

```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- bash
```

If `bash` is unavailable, try `sh` instead.

### 11.2 Open ClickHouse Client

From inside the pod shell:

```bash
clickhouse-client
```

You can now run SQL queries interactively. Use `exit` to leave the client, and another `exit` to leave the pod shell.

### 11.3 Confirm the OTel table exists

Command run from the host terminal:

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "EXISTS TABLE observability.otel_logs"
```

Observed output:

```text
1
```

`1` confirms that the table exists. It does **not** confirm that rows have been inserted.

### 11.4 Count inserted rows

Run:

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT count() FROM observability.otel_logs"
```

Interpretation:

- `0`: no rows are currently present; inspect receiver activity and exporter errors.
- A value greater than `0`: the table contains rows; inspect sample records next.

**This check remains pending in the recorded setup. Do not mark ingestion as fully verified until the count is positive and sample records look valid.**

### 11.5 Inspect sample records

After confirming rows exist, run:

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT * FROM observability.otel_logs LIMIT 5"
```

Review the actual column names and values. The OTel exporter's schema may differ from Fluent Bit's custom schema, so do not assume the two tables have identical columns.

### 11.6 Check the newest timestamp

First inspect the table schema:

```sql
DESCRIBE TABLE observability.otel_logs;
```

Then query the timestamp column using its actual name. For example, if the column is `Timestamp`:

```sql
SELECT max(Timestamp) FROM observability.otel_logs;
```

## 12. Compare Fluent Bit and OTel ingestion

Destinations:

| Collector | Destination table |
|---|---|
| Fluent Bit | `observability.kubernetes_logs` |
| OTel Collector DaemonSet | `observability.otel_logs` |

Count Fluent Bit rows:

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT count() FROM observability.kubernetes_logs"
```

Count OTel rows:

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT count() FROM observability.otel_logs"
```

Run the checks close together and compare records from a common time window. Total counts may differ because of parsing, filtering, collector start position, retry behavior, metadata handling, or duplicate collection. The rendered OTel file receiver configuration used `start_at: end`, so it begins reading at the end of existing files rather than replaying all historical content.

For a fair comparison, validate that both tables contain records from the same workloads and period. Row counts alone do not establish that one collector is better.

## 13. Troubleshooting

### Table exists but row count is zero

1. Inspect OTel agent logs for export, connection, authentication, or schema errors.
2. Confirm the file receiver is watching `/var/log/pods` files.
3. Confirm the pod log directories are mounted in the OTel agent.
4. Confirm the ClickHouse endpoint and port are correct and reachable.
5. Inspect the actual table schema and exporter configuration.

Example:

```bash
kubectl logs -n monitoring otel-logs-opentelemetry-collector-agent-gzx4w --since=10m
```

### Check pod status

```bash
kubectl get pods -n monitoring -o wide
```

### Check Helm releases

```bash
helm list -n monitoring
```

### Check table schema

```sql
DESCRIBE TABLE observability.otel_logs;
```

### Operational cautions

- Do not disable or remove Fluent Bit while validating the OTel path.
- Do not change the existing `otel-collector` release unless there is a separate reviewed requirement.
- Keep destinations separate during the comparison.
- For production, configure resource requests/limits, review credentials and permissions, decide on checkpoint persistence, and validate retention/storage behavior.
- `create_schema: true` is a convenience, not a substitute for inspecting and validating the created schema.

## 14. Progress checklist

| Task | Status |
|---|---|
| Existing Fluent Bit deployment identified | Complete |
| Fluent Bit destination identified as `observability.kubernetes_logs` | Complete |
| Existing OTel metrics/traces Collector reviewed | Complete |
| Separate OTel logs values file created | Complete |
| Helm chart rendered successfully | Complete |
| New `otel-logs` Helm release installed | Complete |
| Two OTel logs agent pods observed running | Complete |
| Collector startup and file receiver activity checked | Complete |
| `observability.otel_logs` existence checked (`1`) | Complete |
| OTel logs row count checked | Pending |
| Sample OTel log records inspected | Pending |
| Ingestion comparison over a common time window | Pending |
| Grafana comparison dashboard | To do after evidence is collected |
| Final comparison README/evidence review | To do after verification |

## 15. Quick command reference

```bash
# Check current Kubernetes context
kubectl config current-context

# Check OTel logs agents
kubectl get pods -n monitoring -o wide | grep otel-logs

# Inspect OTel logs agent output
kubectl logs -n monitoring otel-logs-opentelemetry-collector-agent-gzx4w --since=5m

# List Helm releases in monitoring
helm list -n monitoring

# Check whether the OTel table exists
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "EXISTS TABLE observability.otel_logs"

# Count OTel log rows
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT count() FROM observability.otel_logs"

# Count Fluent Bit log rows
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT count() FROM observability.kubernetes_logs"
```

## Conclusion

The separate OTel logs Collector DaemonSet is deployed, and the `observability.otel_logs` table exists. The next step is to verify the table's row count and inspect sample records. Only after confirming actual records should ingestion be marked fully verified and the data used as evidence for the Fluent Bit versus OTel comparison.
