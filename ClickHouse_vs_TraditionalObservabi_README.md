# ClickHouse vs Traditional Observability Stack
## POC Comparison — Architecture, Validation, Measurements, and Limitations

**Environment:** Local Kubernetes Kind cluster (`kind-kind-clickhouse-poc`)  
**Purpose:** Compare a ClickHouse-centered observability design with a traditional multi-backend design  
**Status:** Functional checks completed for selected paths; Grafana datasource troubleshooting and controlled benchmarking remain follow-up work  

> **Important:** This is a POC report, not a production benchmark. Results are environment-, workload-, and time-specific. Multiple snapshots collected at different times are kept separate rather than silently combined.

---

## 1. Executive Summary

This POC explores two approaches to storing and querying observability telemetry.

1. **ClickHouse-centered stack:** ClickHouse is the main database for logs and configured OpenTelemetry metrics/traces. ClickHouse Keeper provides coordination. Fluent Bit sends Kubernetes container logs to ClickHouse, while the ClickHouse-side OpenTelemetry Collector exports configured metrics and traces.
2. **Traditional multi-backend stack:** VictoriaMetrics stores metrics, Loki stores logs, and Tempo stores traces. `vmagent` scrapes metrics and writes them to VictoriaMetrics, Fluent Bit sends logs to Loki, and the traditional OpenTelemetry Collector sends traces to Tempo.

Grafana is the visualization layer. It queries the configured data sources; it does not itself store these backends' telemetry.

### What was verified in the POC

- ClickHouse accepted a simple SQL query.
- Loki and Tempo readiness endpoints returned `ready`.
- VictoriaMetrics health endpoint returned HTTP `200 OK` and body `OK`.
- A near-time five-minute log-count check returned **65 records** in both ClickHouse and Loki.
- Recent backend-reported query execution samples were in the low-millisecond range, but their timing definitions are not identical.
- Resource-usage snapshots were collected with `kubectl top pods`.
- Test traces were retrievable from ClickHouse and Tempo; a shared trace ID and matching identifying attributes were found in both backends.

### What has not been proved

This POC does **not** prove that either architecture is universally faster, cheaper, or more storage-efficient. Workloads, pipelines, query timing definitions, storage scopes, and sampling times were not fully standardized. A fair verdict requires a controlled same-workload benchmark.

---

## 2. What Observability Means

Observability helps engineers understand a system through three complementary types of telemetry:

| Type | What it tells us | Example |
|---|---|---|
| Logs | What happened as individual events | A pod reports an error |
| Metrics | How much or how often something happens | CPU usage or request rate |
| Traces | How a request moves across components | A request calls a service and then a database |

A practical workflow might be: a dashboard shows high latency, a metric identifies the affected service, logs show the error, and a trace reveals which operation took the time.

---

## 3. POC Environment and Components

The Kubernetes context used in the POC was `kind-kind-clickhouse-poc`. Grafana runs separately from the Kind cluster.

### 3.1 ClickHouse-side namespaces

| Namespace | Components / responsibility |
|---|---|
| `clickhouse` | ClickHouse server and ClickHouse Keeper |
| `monitoring` | ClickHouse-side OpenTelemetry Collector and metric exporters such as node exporter and kube-state-metrics |
| `logging` | ClickHouse-side Fluent Bit, which reads Kubernetes container logs and sends them to ClickHouse |

### 3.2 Traditional observability namespace

Namespace: `traditional-observability`

| Component | Responsibility |
|---|---|
| Fluent Bit (2 pods) | Collects Kubernetes logs |
| Loki | Stores and queries logs |
| Loki storage exporter | Exposes Loki storage-related metrics |
| OpenTelemetry Collector | Sends configured traces to Tempo; logs/metrics pipelines were configured to use a debug exporter |
| Tempo | Stores and queries traces |
| VictoriaMetrics | Stores metrics |
| vmagent | Scrapes configured metric targets and forwards samples |

### 3.3 Important terminology

- **Pod:** A Kubernetes unit that runs one or more containers.
- **Namespace:** A logical grouping of Kubernetes resources.
- **Datasource:** A connection Grafana uses to query a backend.
- **Exporter:** A component that sends telemetry to a destination.
- **Scrape:** A collector periodically requests metrics from an endpoint.
- **Ingestion:** Accepting and storing incoming telemetry.
- **Retention:** How long telemetry is kept.
- **Cardinality:** The number of unique time-series/label combinations; high cardinality can increase resource use.

---

## 4. Architecture A — ClickHouse-Centered Observability

### 4.1 ClickHouse

ClickHouse is a column-oriented analytical database. In this POC, the `observability` database contains tables for Kubernetes logs and OpenTelemetry telemetry, including:

- `observability.kubernetes_logs`
- `observability.otel_metrics_gauge`
- `observability.otel_metrics_sum`
- `observability.otel_metrics_summary`
- `observability.otel_metrics_histogram`
- `observability.otel_traces`

The `kubernetes_logs` table is used for the log-count comparison.

### 4.2 ClickHouse Keeper

Keeper provides coordination for ClickHouse deployments that require it, including replicated/distributed setups. It runs as a separate pod. Grafana queries ClickHouse; Keeper is not the Grafana datasource.

### 4.3 Fluent Bit log path

The ClickHouse-side Fluent Bit DaemonSet reads files matching `/var/log/containers/*.log`, enriches records with Kubernetes metadata, and sends them over HTTP port `8123`.

Observed target:

```text
clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:8123
```

Observed destination table:

```text
observability.kubernetes_logs
```

The configured output uses JSONEachRow/json-lines formatting and adds a timestamp field.

**Important:** Kubernetes log rows in `observability.kubernetes_logs` arrive through Fluent Bit. The ClickHouse-side Collector's logs pipeline was configured with a debug exporter, so it is not the pipeline writing those rows.

### 4.4 OpenTelemetry Collector

The ClickHouse-side Collector exports configured metrics and traces to ClickHouse. Observed settings included `async_insert: true` and `create_schema: true`. The logs pipeline used a debug exporter.

### 4.5 Simplified data path

```text
Kubernetes container log files
          |
          v
    Fluent Bit (logging)
          |
          v
  ClickHouse HTTP :8123
          |
          v
 observability.kubernetes_logs

Metrics endpoints --> ClickHouse-side OTel Collector --> ClickHouse
Application traces --> ClickHouse-side OTel Collector --> ClickHouse

Grafana --> ClickHouse datasource
```

The actual telemetry routes depend on the receiver, pipeline, and exporter configuration.

---

## 5. Architecture B — Traditional Multi-Backend Observability

### 5.1 VictoriaMetrics — metrics

`vmagent` scrapes configured targets and sends samples to VictoriaMetrics. Observed scrape jobs included node exporter, kube-state-metrics, cAdvisor through the Kubernetes node proxy, Loki metrics, VictoriaMetrics self-metrics, Fluent Bit metrics, and the Loki storage exporter. The configured global scrape interval was 30 seconds.

Observed in-cluster service endpoint:

```text
victoria-metrics-victoria-metrics-single-server.traditional-observability.svc.cluster.local:8428
```

### 5.2 Loki — logs

Fluent Bit sends logs to Loki and attaches labels such as `namespace`, `pod`, and `container`. LogQL can select logs by labels and count entries over a time window.

### 5.3 Tempo — traces

The traditional OpenTelemetry Collector sends traces to Tempo over OTLP gRPC at:

```text
tempo.traditional-observability.svc.cluster.local:4317
```

Tempo's query API was exposed on port `3200`.

### 5.4 Traditional Collector configuration caveat

The observed traditional Collector configuration used:
- traces pipeline → Tempo through OTLP gRPC
- logs pipeline → debug exporter
- metrics pipeline → debug exporter

In this setup, Fluent Bit sends Kubernetes logs to Loki, and `vmagent` sends scraped metrics to VictoriaMetrics. Do not describe the traditional Collector as the component writing all logs and metrics to the traditional backends.

### 5.5 Simplified data path

```text
Metrics endpoints --> vmagent --> VictoriaMetrics
Kubernetes logs --> Fluent Bit --> Loki
Application traces --> OTel Collector --> Tempo

Grafana --> VictoriaMetrics / Loki / Tempo
```

---

## 6. Grafana and Datasources

Grafana is the shared visualization and exploration layer. It queries each configured datasource.

| Datasource name | Type | Purpose |
|---|---|---|
| `grafana-clickhouse-datasource-1` | ClickHouse | Query ClickHouse SQL tables |
| `loki-1` | Loki | Query logs with LogQL |
| `VM-1` | Prometheus-compatible | Query VictoriaMetrics through its compatible API |
| `tempo` | Tempo | Search/open traces |

Other datasource entries were also present in Grafana; confirm the connection settings for the datasource used by each panel.

### Networking note

During troubleshooting, datasource URLs included loopback addresses such as `127.0.0.1:8123`, `127.0.0.1:32446`, `127.0.0.1:31720`, and `127.0.0.1:31000`. `127.0.0.1` refers to the same network namespace as the client. If Grafana runs on a separate server, its loopback is not the laptop's loopback or a Kind pod's loopback. The endpoint must be reachable from the Grafana server.

A later Grafana request returned:

```text
POST "http://127.0.0.1:31720/api/v1/query_range": EOF
```

The root cause was not confirmed when this report was prepared. Check the `VM-1` URL, service/NodePort mapping, reachability from the Grafana host, and **Save & test** result before trusting the panel.

---

## 7. Health Checks

These basic health checks were observed during the POC.

### ClickHouse

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client --query "SELECT 1"
```

Observed output: `1`.

Meaning: the client connected to ClickHouse and the server executed a simple SQL query.

### Loki

```bash
kubectl exec -n traditional-observability loki-stack-0 -- \
  wget -qO- http://localhost:3100/ready
```

Observed output: `ready`.

### VictoriaMetrics

```bash
kubectl exec -n traditional-observability \
  victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -S -O- http://victoria-metrics-victoria-metrics-single-server:8428/health
```

Observed response: HTTP `200 OK`, body `OK`.

### Tempo

```bash
kubectl exec -n traditional-observability tempo-0 -- \
  wget -qO- http://localhost:3200/ready
```

Observed output: `ready`.

A healthy endpoint alone does not prove that all telemetry is ingested, complete, or queryable from Grafana; ingestion and query checks are also required.

---

## 8. Log Count Comparison

The test compares entries in the `clickhouse` namespace over a rolling five-minute window.

### 8.1 ClickHouse SQL

```sql
SELECT
    now() AS checked_at,
    now() - INTERVAL 5 MINUTE AS window_start,
    count() AS log_count
FROM observability.kubernetes_logs
WHERE namespace = 'clickhouse'
  AND timestamp >= now() - INTERVAL 5 MINUTE
  AND timestamp <= now()
FORMAT PrettyCompact;
```

One observed result at `2026-10-12 07:29:50` was **65 logs**, for the five-minute window beginning at `07:24:50`.

### 8.2 Loki LogQL

```logql
sum(count_over_time({namespace="clickhouse"}[5m]))
```

A near-time response returned:
- count: `65`
- `execTime`: `0.002746459` seconds (about `2.75 ms`)
- `queueTime`: `0.006749964` seconds (about `6.75 ms`)

Three immediately repeated Loki requests earlier returned a count of `66`, with execution times of approximately `5.11 ms`, `4.44 ms`, and `4.51 ms`.

### 8.3 Interpretation

The near-time checks returned 65 in both systems. This is useful evidence that both paths had a similar number of matching records at that time. Equal counts do not prove that the underlying records are identical. The queries ran sequentially and the rolling window continuously changes. Record-level equality requires stable unique IDs or a controlled test batch and a comparison of IDs.

---

## 9. Query Latency Observations

### 9.1 ClickHouse server-side query log

The following query was used to inspect recent ClickHouse query-log samples:

```sql
SELECT
    event_time,
    query_duration_ms,
    read_rows,
    read_bytes
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%FROM observability.kubernetes_logs%'
  AND query LIKE '%INTERVAL 5 MINUTE%'
  AND event_time >= now() - INTERVAL 10 MINUTE
ORDER BY event_time DESC
LIMIT 10
FORMAT PrettyCompact;
```

Recorded samples:

| Event time | Query duration | Rows read | Bytes read |
|---|---:|---:|---:|
| 07:28:24 | 9 ms | 8,443 | 231,749 |
| 07:28:23 | 5 ms | 8,437 | 231,542 |
| 07:28:08 | 5 ms | 8,348 | 229,024 |

Mean of these three samples: approximately **6.3 ms**. Another query at `07:29:50` was recorded at 15 ms, reading 8,836 rows and 242,799 bytes. Individual query times varied.

### 9.2 Loki timings

Three recorded Loki `execTime` samples were:
- 5.108 ms
- 4.436 ms
- 4.513 ms

Mean `execTime`: approximately **4.69 ms**.

A later response reported `execTime` of 2.75 ms and `queueTime` of 6.75 ms.

### 9.3 Limitation

ClickHouse `query_duration_ms` is a server-side query-log field. Loki reports its own `execTime` and `queueTime`. The requests were not a formal benchmark with a fixed dataset, synchronized window, controlled cache state, and many repeated trials. Network/client overhead is not represented identically.

**Safe conclusion:** Both backends returned this count query with low-millisecond backend-reported execution times in the observed samples. These measurements are not equivalent enough to declare a latency winner.

---

## 10. CPU and Memory — Keep Snapshots Separate

Resource usage was collected with `kubectl top pods`. A local Kind kubelet certificate issue was previously worked around by enabling `--kubelet-insecure-tls` for Metrics Server. This is a local POC workaround only; do not copy that insecure setting into production.

### 10.1 Earlier snapshot recorded in the detailed POC notes

| Component | CPU | Memory |
|---|---:|---:|
| ClickHouse | 113m | 716 MiB |
| ClickHouse Keeper | 18m | 125 MiB |
| **ClickHouse + Keeper total** | **131m** | **841 MiB** |
| Traditional Fluent Bit pod 1 | 2m | 7 MiB |
| Traditional Fluent Bit pod 2 | 1m | 9 MiB |
| Loki | 6m | 87 MiB |
| Loki storage exporter | 2m | 16 MiB |
| Traditional OTel Collector | 3m | 76 MiB |
| Tempo | 3m | 113 MiB |
| VictoriaMetrics | 7m | 144 MiB |
| vmagent | 3m | 29 MiB |
| **Traditional stack total** | **approximately 27m** | **481 MiB** |

The source notes contain inconsistent arithmetic for the displayed traditional CPU total; adding the listed rounded component values gives 27m. Treat CPU totals as approximate because pod values are rounded.

### 10.2 Later terminal snapshot collected on 12 October 2026

Commands:

```bash
kubectl top pods -n clickhouse
kubectl top pods -n traditional-observability
```

ClickHouse namespace:

| Pod | CPU | Memory |
|---|---:|---:|
| `clickhouse-clickhouse-0-0-0` | 142m | 454 Mi |
| `clickhouse-keeper-keeper-0-0` | 20m | 127 Mi |
| **Total** | **162m** | **581 Mi** |

Traditional namespace:

| Pod | CPU | Memory |
|---|---:|---:|
| `fluent-bit-traditional-2pncc` | 2m | 7 Mi |
| `fluent-bit-traditional-gwr68` | 2m | 6 Mi |
| `loki-stack-0` | 5m | 75 Mi |
| `loki-storage-exporter-747469dbb-vbh5s` | 3m | 19 Mi |
| `otel-collector-traditional-opentelemetry-collector-7554845tr2qw` | 4m | 38 Mi |
| `tempo-0` | 5m | 91 Mi |
| `victoria-metrics-victoria-metrics-single-server-0` | 8m | 151 Mi |
| `vmagent-victoria-metrics-agent-595f9665b4-6m5zm` | 4m | 41 Mi |
| **Total** | **33m** | **428 Mi** |

### 10.3 How to interpret the snapshots

In the later snapshot, the ClickHouse namespace used **153 MiB more memory** than the traditional namespace (581 MiB vs 428 MiB), approximately **35.7% more relative to the traditional stack**. The earlier snapshot recorded different values (841 MiB vs 481 MiB). These snapshots were taken at different times and must not be averaged or treated as a single measurement.

These results do not prove that one architecture is generally less efficient: the workloads and telemetry rates were not controlled, components have different responsibilities, and this is a short sample rather than a time average or peak measurement.

For a fair comparison, sample both stacks repeatedly over the same interval under the same incoming workload, and record requests/limits, restart counts, ingestion lag, and dropped records.

### 10.4 Grafana memory panel recommendation

For stack-level memory comparison, use the same datasource and metric for both namespaces. The proposed `VM-1` PromQL queries are:

**ClickHouse namespace:**

```promql
sum(
  container_memory_working_set_bytes{
    namespace="clickhouse",
    container!="",
    image!=""
  }
) / 1024 / 1024
```

**Traditional namespace:**

```promql
sum(
  container_memory_working_set_bytes{
    namespace="traditional-observability",
    container!="",
    image!=""
  }
) / 1024 / 1024
```

These queries require `container_memory_working_set_bytes` to be present in VictoriaMetrics and the Grafana datasource to be reachable. At document preparation time, Grafana's request to `http://127.0.0.1:31720/api/v1/query_range` returned `EOF`, and connectivity had not yet been confirmed. Therefore, do not treat these panel results as validated until the datasource issue is resolved.

A ClickHouse SQL query separately returned `CGroupMemoryUsed = 412.04 MiB` in one check. That metric is not directly equivalent to `container_memory_working_set_bytes` and represents the ClickHouse server container, not the full ClickHouse namespace.

---

## 11. Ingestion Counter Observations

### 11.1 ClickHouse process-wide insert counters

Query used:

```sql
SELECT
    event,
    value
FROM system.events
WHERE event IN ('InsertQuery', 'InsertedRows', 'InsertedBytes')
ORDER BY event
FORMAT PrettyCompact;
```

First snapshot:
- `InsertQuery`: 24,082
- `InsertedRows`: 14,026,999
- `InsertedBytes`: 6,418,256,742

Second snapshot:
- `InsertQuery`: 24,167
- `InsertedRows`: 14,075,102
- `InsertedBytes`: 6,440,548,892

Differences:
- 85 insert queries
- 48,103 rows
- 22,292,150 bytes

If the interval was approximately 60 seconds, the approximate rates would be 1.42 insert queries/second, 802 rows/second, and 371.5 KB/second (decimal bytes).

**Scope warning:** These counters are process-wide; they are not guaranteed to represent only Kubernetes logs or only one POC table.

### 11.2 VictoriaMetrics storage counter

Query used:

```bash
kubectl exec -n traditional-observability \
  victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -qO- \
  'http://victoria-metrics-victoria-metrics-single-server:8428/api/v1/query?query=vm_rows_added_to_storage_total'
```

Observed samples:
- Unix timestamp `1791790429`: `9,655,167`
- Unix timestamp `1791790514`: `9,704,965`

Difference: 49,798 rows over 85 seconds, or approximately 586 rows/second.

### 11.3 Why these rates are not directly comparable

ClickHouse `InsertedRows` and VictoriaMetrics `vm_rows_added_to_storage_total` have different definitions and scopes. The former counts rows inserted into the ClickHouse process; the latter counts rows added to VictoriaMetrics storage. They may include different sources and telemetry types. These are useful observations, not proof that one backend has higher ingestion throughput.

---

## 12. Storage Observations

Storage measurements were captured at different times and with different measurement scopes.

### 12.1 Earlier detailed POC snapshot

ClickHouse `system.parts` observations:
- `otel_metrics_gauge`: about 1.02 GiB, around 101.0 million rows
- `otel_metrics_sum`: about 652 MiB, around 58.3 million rows
- `kubernetes_logs`: around 62–65 MB, about 3.76 million rows
- `otel_metrics_summary`: about 1.3 MB, around 49.8 thousand rows
- `otel_metrics_histogram`: under 0.5 MB, around 12.3 thousand rows
- `otel_traces`: 3 rows, about 19 KiB
- `otel_traces_trace_id_ts`: 3 rows, about 3.8 KiB

Observed directory usage:
- ClickHouse `/var/lib/clickhouse`: about 3.1 GB
- VictoriaMetrics `/storage`: around 98.6 MB in one check
- Loki `/data`: around 82.8 MB in one check
- Tempo `/var/tempo`: around 76 KB at the time measured

PVC requests recorded:
- ClickHouse: 5 Gi
- ClickHouse Keeper: 2 Gi
- VictoriaMetrics: 16 Gi
- Loki: 5 Gi
- Tempo: 5 Gi

### 12.2 Additional ClickHouse storage observations

Other checks during the POC reported:
- ClickHouse PVC directory usage: about 2.8 GiB in one check, later around 3.2 GiB.
- Loki data directory usage: about 66–73 MiB in earlier checks.
- ClickHouse `observability` database: about 1.47 GiB in a recorded `system.parts` breakdown.
- ClickHouse `observability.kubernetes_logs`: 58.17 MiB and 3,598,050 rows in a recorded check.

These figures are from different snapshots and should not be combined as if measured simultaneously.

### 12.3 Storage interpretation

PVC requested capacity is not the same as used storage. ClickHouse table parts, database size, full PVC directory usage, and Loki data-directory usage have different scopes. Data volume, retention, compression, indexing, compaction, and workload were not held constant. Therefore, these figures do not establish a storage-efficiency winner.

---

## 13. Trace Verification

The POC sent test traces to both paths and checked whether they could be queried back.

### ClickHouse trace

A fresh trace was found in `observability.otel_traces`:
- Span name: `clickhouse-fresh-test`
- Service name: `clickhouse-trace-verification`
- Attribute: `test.run=fresh-verification`

### Tempo trace

Tempo's trace and search APIs returned test traces, including `tempo-ingestion-verification`.

### Shared trace ID check

Shared trace ID:

```text
64126405d0a7d5af2f83dcf6660f3e71
```

It was found in both ClickHouse and Tempo with matching identifying information:
- Span name: `shared-comparison-span`
- Service name: `shared-trace-comparison`
- Attributes included `comparison.id=shared-trace-001` and `comparison.test=same-span-both-backends`

The span IDs differed because the test-generation methods were not identical at the span level. The supported claim is that the same trace ID and matching identifying attributes were found in both backends; the exact same span record was not proven.

---

## 14. Reliability and Operational Trade-offs

A comparison should consider more than query latency:

- How many services need installation, upgrades, and backups?
- How are retention and deletion configured?
- How are missing records troubleshot?
- How are authentication, authorization, and network access secured?
- What happens if a component or node fails?
- How easy are dashboards and alerts to maintain?
- Can logs, metrics, and traces be correlated reliably?
- What are the backup, restore, and disaster recovery procedures?
- What are the CPU/memory requests and limits?
- What are ingestion lag and dropped-record rates?

### ClickHouse-centered approach

**Potential benefit:** Logs, metrics, and traces can be queried in one database/query environment when schemas and ingestion are designed appropriately.

**Operational considerations:** Schema design, SQL, storage sizing, merges, replication, and Keeper/cluster operations need to be understood and maintained.

### Traditional multi-backend approach

**Potential benefit:** Separate systems can be selected and tuned for their telemetry type, with their own APIs and query languages.

**Operational considerations:** Multiple backends need their own capacity planning, retention, upgrades, troubleshooting, and access configuration. Cross-signal correlation and dashboard consistency require deliberate configuration.

These are architectural trade-offs, not measured wins from this small POC.

---

## 15. Grafana Dashboard Plan

Suggested panel order:

1. **ClickHouse Logs — Last 5 Minutes** — Stat panel using ClickHouse SQL.
2. **Loki Logs — Last 5 Minutes** — Stat panel using LogQL.
3. **ClickHouse vs Loki — Log Count Comparison** — label clearly that counts were near-time checks, not record-level proof.
4. **ClickHouse Query Duration** — query `system.query_log`; verify the ClickHouse datasource.
5. **Loki Query Execution Time** — use query statistics if supported; otherwise keep the timing as a recorded benchmark observation, not a live metric.
6. **VictoriaMetrics Targets Up** — Prometheus-compatible query such as `sum(up)`, if the metric is available.
7. **VictoriaMetrics Rows Added/sec** — `rate(vm_rows_added_to_storage_total[5m])`, if the metric is available.
8. **ClickHouse vs Traditional — CPU Usage** — same datasource and metric definition for both namespaces.
9. **ClickHouse vs Traditional — Memory Usage** — same datasource and metric definition for both namespaces.
10. **Loki Logs Explorer** — `{namespace="clickhouse"}`.
11. **Tempo Trace Search** — search a known service or trace ID.
12. **POC Notes / Limitations** — text panel describing benchmark scope and caveats.

---

## 16. Controlled Benchmark Plan

For a defensible comparison:

1. Generate a repeatable test workload with known log, metric, and trace counts.
2. Send the same data and equivalent labels/attributes through both paths.
3. Synchronize test start and end times.
4. Use semantically equivalent queries and equivalent time windows.
5. Run multiple trials and record p50/p95 latency, not only one average.
6. Separate client round-trip time, queue time, and backend execution time.
7. Measure ingestion lag and verify that records were not dropped.
8. Compare stable record IDs; matching counts alone are insufficient.
9. Sample CPU, memory, and disk I/O regularly under the same workload.
10. Compare storage growth after equal data volume and equal retention.
11. Record component versions, configuration, resource limits, and test duration.
12. Test both warmed-cache and cold-start behavior if relevant.

Only after this should the team make a defensible performance or cost recommendation.

---


## 17. Final Conclusion

The POC demonstrated that the tested telemetry paths can return selected logs, metrics, and traces. It captured baseline observations for query execution, resource use, ingestion counters, and storage.

The evidence supports these statements:
- Both paths were operational for the tested checks.
- A near-time five-minute log-count check returned 65 in both systems.
- The sampled backend-reported query execution times were in the low-millisecond range.
- Resource, ingestion-counter, and storage observations were collected.
- Test traces were retrievable in both backends, including a shared trace ID with matching identifying attributes.

The evidence does **not** support claiming that ClickHouse is universally faster, cheaper, more resource-efficient, or more storage-efficient than the multi-backend stack. A reliable conclusion requires equivalent workloads, consistent measurement definitions, repeated trials, and verified Grafana queries.

**POC status:** Functional validation completed for selected checks; Grafana datasource troubleshooting and controlled benchmark remain follow-up work.
