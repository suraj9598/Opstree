# ClickHouse vs Traditional Observability Stack — Comparison

## 1. Objective

This document records only the comparison work performed between a ClickHouse-based observability backend and a traditional observability stack for Kubernetes telemetry. It excludes installation and deployment walkthroughs.

The goal is to evaluate:
- Log storage footprint
- Log ingestion coverage and freshness
- Query latency
- CPU and memory usage
- Operational considerations and Grafana integration

The comparison must use documented commands, repeatable measurements, and clearly stated measurement scopes. No backend should be declared the winner based on a single metric or a non-equivalent measurement.

## 2. Environments and Architecture

### ClickHouse-based stack

- Kubernetes context: `kind-kind-clickhouse-poc`
- Namespace: `clickhouse`
- ClickHouse server pod: `clickhouse-clickhouse-0-0-0`
- ClickHouse Keeper pod: `clickhouse-keeper-keeper-0-0`
- Logs table: `observability.kubernetes_logs`
- Metrics tables: `observability.otel_metrics_*`
- Traces table: `observability.otel_traces`
- Grafana datasource: `grafana-clickhouse-datasource-1`

Fluent Bit sends Kubernetes logs to ClickHouse. The OpenTelemetry Collector exports metrics and traces to ClickHouse. ClickHouse Operator manages the ClickHouse deployment, and ClickHouse Keeper provides coordination.

### Traditional observability stack

- Namespace: `traditional-observability`
- Log backend: Loki
- Metrics backend: VictoriaMetrics
- Trace backend: Tempo
- Grafana datasources: `loki-1`, `VM-1`, `tempo`

Fluent Bit sends logs to Loki. `vmagent` scrapes metrics and sends them to VictoriaMetrics. OpenTelemetry Collector sends traces to Tempo.

### Important scope note

This document records measurements observed during the POC. The stacks do not necessarily have identical component scopes, data retention, filesystem accounting, query semantics, or timing methods. Any difference must be called out before interpreting a result as a fair performance comparison.

## 3. Current Side-by-Side Results

**Status as of 2026-10-13. These are observed samples, not final benchmark conclusions.**

| Metric | ClickHouse stack | Traditional stack | Status / interpretation |
|---|---:|---:|---|
| Log storage measurement | 67.34 MiB | 91.9 MiB | Provisional; scopes and measurement methods differ |
| Recent log count, approximately one-hour query | 16,644–16,744 across observed runs | 16,647–16,910 across observed runs | Counts are close in selected samples; query windows were not perfectly synchronized |
| Query latency sample median | 2 ms | 17.393 ms | Not an apples-to-apples comparison; timers measure different scopes |
| CPU usage | Existing Grafana panel has provisional samples | Existing Grafana panel has provisional samples | Revalidate using equivalent component scope and sampling window |
| Memory usage | Existing Grafana panel has provisional samples | Existing Grafana panel has provisional samples | Revalidate using equivalent component scope and sampling window |

Do not calculate a final storage, latency, CPU, or memory winner from this table yet.

## 4. Storage Measurement

### 4.1 ClickHouse logs table

The following query was used to inspect active parts for the Kubernetes logs table:

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    sum(rows) AS total_rows,
    count() AS active_parts
FROM system.parts
WHERE active
  AND database = 'observability'
  AND table = 'kubernetes_logs'
GROUP BY database, table;
```

Observed result:
- Active table-part disk size: **67.34 MiB**
- Compressed data bytes: **67.28 MiB**
- Uncompressed data bytes: **1.14 GiB**
- Rows: **3,945,040** at the time of this query
- Active parts: **9**

Counts and sizes can change while ingestion continues. Re-run the query when producing a final report and record the timestamp.

### 4.2 Loki data directory

The custom Loki storage exporter walks the mounted `/loki-data` directory and sums each file's `stat().st_size`. It does not measure allocated filesystem blocks in the same way as every possible disk-usage command.

Commands used:

```bash
kubectl exec -n traditional-observability loki-storage-exporter-747469dbb-vbh5s -- \
  sh -c 'du -sh /loki-data 2>/dev/null; find /loki-data -type f 2>/dev/null | wc -l'
```

Observed result:
- `/loki-data` directory usage: **91.9M** (as reported by `du`)
- Number of files: **2,862**

The exporter metric represents the sum of file logical sizes under `/loki-data`. Confirm the exporter value and filesystem accounting method before finalizing the storage comparison.

### 4.3 Storage interpretation

The current values are indicative only:
- ClickHouse: active parts for the `observability.kubernetes_logs` table.
- Loki: files in the mounted Loki data directory.

These scopes may include different metadata and internal structures. Do not describe the result as an exact compression benchmark. A fairer comparison should define whether the target is logs-only table/data storage or total backend persistent-storage footprint, then measure equivalent scopes for both systems.

## 5. Log Count and Coverage Validation

### 5.1 Loki labels

Command:

```bash
kubectl exec -n traditional-observability loki-stack-0 -- \
  wget -qO- http://localhost:3100/loki/api/v1/labels
```

Observed labels:

```json
{"status":"success","data":["container","job","namespace","pod"]}
```

### 5.2 Loki count query

A port-forward was used to query Loki from the local machine.

Terminal 1:

```bash
kubectl port-forward -n traditional-observability pod/loki-stack-0 3100:3100
```

Terminal 2:

```bash
curl -G 'http://localhost:3100/loki/api/v1/query' \
  --data-urlencode 'query=sum(count_over_time({namespace=~".+"}[1h]))'
```

Observed counts varied across samples as logs continued to arrive. Examples included **16,637**, **16,647**, **16,739**, **16,732**, and **16,910**.

The query counts matching entries across all namespaces with a non-empty `namespace` label over the query's one-hour range. Query evaluation time advances between runs.

### 5.3 ClickHouse count query

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client --query "
SELECT
    count() AS total_logs,
    min(timestamp) AS first_log,
    max(timestamp) AS latest_log
FROM observability.kubernetes_logs
WHERE timestamp >= now() - INTERVAL 1 HOUR
"
```

Observed examples:
- 16,612 rows; first timestamp `2026-10-13 06:04:59`; latest timestamp `2026-10-13 07:04:54`
- 16,644 rows; first timestamp `2026-10-13 06:06:24`; latest timestamp `2026-10-13 07:06:20`

The ClickHouse query counts rows in `observability.kubernetes_logs` with timestamps in the last hour. This is not automatically identical to Loki's count semantics. To make a rigorous comparison, align the time range explicitly and confirm matching namespace/stream filters and duplicate handling.

### 5.4 Interpretation

The observed counts were close in several samples, which is encouraging, but does not by itself prove exact parity. The queries were run at different instants, and Loki stream/label selection and ClickHouse table row semantics may differ. For final validation:
1. Choose an explicit fixed start and end timestamp.
2. Apply equivalent namespace and workload filters.
3. Run each query against that same interval.
4. Record returned counts, first/last timestamps, and query time.
5. Repeat the test and note ingestion lag or duplicate semantics if relevant.

## 6. Query Latency Measurements

### 6.1 ClickHouse client timer

Initial one-off command:

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client --time --query "
SELECT count()
FROM observability.kubernetes_logs
WHERE timestamp >= now() - INTERVAL 1 HOUR
"
```

Observed five runs:

| Run | Returned count | Client-reported duration |
|---:|---:|---:|
| 1 | 16,743 | 2 ms |
| 2 | 16,743 | 3 ms |
| 3 | 16,744 | 2 ms |
| 4 | 16,743 | 2 ms |
| 5 | 16,743 | 2 ms |

- Median: **2 ms**
- Range: **2–3 ms**

`clickhouse-client --time` reports client-side query timing. The query's rolling one-hour window changes between runs, and the count changes as ingestion continues.

A separate shell `time kubectl exec ...` measurement was **137 ms** (`real 0m0.137s`), but this includes Kubernetes exec and process-startup overhead and should not be treated as ClickHouse server-only latency.

### 6.2 Loki HTTP timer

The Loki API was queried five times with `curl`:

```bash
for i in {1..5}; do
  echo "Run $i"

  curl -s -o /tmp/loki-query.json \
    -w 'HTTP total time: %{time_total} seconds\n' \
    -G 'http://localhost:3100/loki/api/v1/query' \
    --data-urlencode \
    'query=sum(count_over_time({namespace=~".+"}[1h]))'

  python3 -c '
import json
with open("/tmp/loki-query.json") as f:
    d = json.load(f)
print("Count =", d["data"]["result"][0]["value"][1])
'
done
```

Observed HTTP total times:
- 0.019006 seconds
- 0.014990 seconds
- 0.021392 seconds
- 0.017393 seconds
- 0.014259 seconds

Derived median: **0.017393 seconds (17.393 ms)**. Range: **14.259–21.392 ms**.

These are client-observed total HTTP request times and include network/request/response overhead. They are not directly equivalent to the ClickHouse `--time` measurement. A prior attempt to read `stats.summary.execTime` failed because `stats` was absent from some Loki API responses; this is a tooling issue, not a query failure.

### 6.3 Latency conclusion

The current samples show:
- ClickHouse client timer median: 2 ms.
- Loki HTTP total-time median: 17.393 ms.

**Do not claim ClickHouse is faster by a specific factor from these numbers.** The measurement scopes differ. For a fair benchmark, use a common client-side measurement boundary for both backends, run the same equivalent count/filter and fixed time range, warm up both systems, collect multiple samples, and report median and spread. Also separate query execution time from end-to-end request latency.

## 7. CPU and Memory

CPU and memory Grafana panels already exist in the `ClickHouse POC` dashboard. Their current values are provisional and should be revalidated before final conclusions.

### CPU

- ClickHouse query uses `observability.otel_metrics_sum` and derives a rate from samples of `container_cpu_usage_seconds_total` for the `clickhouse` namespace.
- Traditional stack uses VictoriaMetrics/PromQL with `rate(container_cpu_usage_seconds_total[5m])` for `traditional-observability`.
- The ClickHouse calculation uses a first/last-sample delta and is not guaranteed to be equivalent to Prometheus `rate()`.
- The selected namespaces do not have identical scope: the ClickHouse query covers ClickHouse namespace containers, while the traditional query covers the traditional stack namespace. The ClickHouse-side collector may run outside the ClickHouse namespace.

Before finalizing, define whether the comparison includes backend pods only or all collection and storage components, use equivalent time windows and filters, and repeat measurements.

### Memory

- ClickHouse panel uses the latest stored `container_memory_working_set_bytes` sample per container from `observability.otel_metrics_gauge`.
- Traditional stack panel uses a VictoriaMetrics query summing `container_memory_working_set_bytes` for containers in `traditional-observability`.
- Values observed in a prior Grafana sample were approximately **789 MiB** for ClickHouse + Keeper and **419 MiB** for the Traditional stack. These are time-specific samples, not final benchmark values.
- A separate `kubectl top pods` snapshot can be used as a sanity check, but it must be captured close to the metric-query time.

For final comparison, ensure equivalent component scope, freshness, units, and sampling windows.

## 8. Grafana Dashboard

Dashboard: **ClickHouse POC**

Existing panels include:
1. ClickHouse vs Loki — Storage Used
2. ClickHouse vs Traditional — CPU Usage
3. ClickHouse vs Traditional — Memory Usage

### Storage panel description

Suggested title: `ClickHouse Logs Table vs Loki Data Storage`

Suggested description:

> Compares on-disk storage for the ClickHouse `observability.kubernetes_logs` table with the Loki data directory measured by the custom storage exporter. ClickHouse reports active table-part storage; Loki reports the total size of files under its mounted data directory. The measurements have different scopes and should be treated as an indicative storage-footprint comparison, not an exact apples-to-apples compression benchmark.

Preserve existing panels and queries unless a validation step demonstrates a specific issue. Record query changes and reasons here when made.


## 8.1. Terminal Resource Snapshot — 2026-10-13

**Command used:**
```bash
kubectl top pods -n clickhouse
kubectl top pods -n traditional-observability
kubectl get pods -n monitoring
```

**ClickHouse namespace output:**
```text
NAME                           CPU(cores)   MEMORY(bytes)
clickhouse-clickhouse-0-0-0    154m         455Mi
clickhouse-keeper-keeper-0-0   18m          127Mi
```

ClickHouse Server + Keeper snapshot total: **172m CPU, 582 MiB memory**.

**Traditional observability namespace output:**
```text
NAME                                                              CPU(cores)   MEMORY(bytes)
fluent-bit-traditional-2pncc                                      2m           6Mi
fluent-bit-traditional-gwr68                                      2m           6Mi
loki-stack-0                                                      3m           91Mi
loki-storage-exporter-747469dbb-vbh5s                             2m           18Mi
otel-collector-traditional-opentelemetry-collector-7554845tr2qw   3m           45Mi
tempo-0                                                           3m           49Mi
victoria-metrics-victoria-metrics-single-server-0                 4m           156Mi
vmagent-victoria-metrics-agent-595f9665b4-6m5zm                   3m           37Mi
```

Traditional listed components snapshot total: **22m CPU, 408 MiB memory**.

**Metrics/supporting pods in `monitoring`:**
- `kube-state-metrics-7cfd7f4c58-2rttv` — Running
- `node-exporter-prometheus-node-exporter-hqqq4` — Running
- `node-exporter-prometheus-node-exporter-lpjrk` — Running
- `node-exporter-prometheus-node-exporter-mgmjv` — Running
- `otel-collector-opentelemetry-collector-59bd9c8d5-4khv7` — Running
- `clickhouse-trace-verify`, `otlp-test`, and `trace-test` — Completed test pods

**Initial interpretation (not a final benchmark):** At this snapshot, ClickHouse Server + Keeper used 582 MiB and 172m CPU; the listed traditional observability components used 408 MiB and 22m CPU. The comparison is not yet fully apples-to-apples: the `monitoring` namespace also contains the OTel Collector and telemetry collectors supporting ClickHouse, and their resource use was not included in the ClickHouse total above. Resource values are point-in-time snapshots and can fluctuate.

**Next validation:** Run `kubectl top pods -n monitoring` to capture current resource usage for the metrics and OTel collection components. Then decide which shared collection components should be included on both sides before drawing a conclusion.



### Monitoring namespace resource snapshot — 2026-10-13

Command:
```bash
kubectl top pods -n monitoring
```

Observed output:
```text
NAME                                                     CPU(cores)   MEMORY(bytes)
kube-state-metrics-7cfd7f4c58-2rttv                      1m           29Mi
node-exporter-prometheus-node-exporter-hqqq4             6m           15Mi
node-exporter-prometheus-node-exporter-lpjrk             4m           16Mi
node-exporter-prometheus-node-exporter-mgmjv             7m           15Mi
otel-collector-opentelemetry-collector-59bd9c8d5-4khv7   12m          140Mi
```

Monitoring namespace listed pods total: **30m CPU, 215 MiB memory**. This is a point-in-time snapshot. The namespace includes the OTel Collector and telemetry collectors used by the ClickHouse POC; before presenting a stack-to-stack total, document whether shared collection infrastructure is included on both sides. Do not simply add all of this namespace to ClickHouse if any components are shared by both backends or collect unrelated cluster-wide metrics.



### Grafana dashboard screenshot snapshot — 2026-10-13, approximately 12:48

Dashboard: `ClickHouse POC`, time range `Last 5 minutes`, auto-refresh `30s`.

Values visible in the screenshot:
- Logs storage panel: Loki **83.55 MiB**, ClickHouse **67.67 MiB**.
- CPU panel: ClickHouse + Keeper **0.228 cores**, Traditional Stack **0.0186 cores**.
- Memory panel: ClickHouse + Keeper **715 MiB**, Traditional Stack **389 MiB**.

These are Grafana panel values at the screenshot time, not the same instant as the earlier `kubectl top pods` terminal snapshots. Do not treat the difference as a query error by itself: panel range, aggregation, scrape timing, and component scope can all affect values. For example, the terminal snapshot showed ClickHouse + Keeper at 172m CPU and 582 MiB memory, while the screenshot shows 228m CPU and 715 MiB. Traditional terminal totals were 22m CPU and 408 MiB memory, while the screenshot shows 18.6m CPU and 389 MiB. These are directionally similar but not identical observations.

**Panel review note:** The CPU and memory panels are currently displaying values and the comparison series are labeled. Before using them for a final benchmark, verify equivalent component scope and metric calculation, and confirm the panel unit/legend labels. Keep this screenshot snapshot separate from terminal measurements in the results table.



### Fresh terminal resource snapshot — 2026-10-13, after Grafana panel review

Commands:
```bash
kubectl top pods -n clickhouse
kubectl top pods -n traditional-observability
kubectl top pods -n monitoring
```

**ClickHouse namespace:**
```text
clickhouse-clickhouse-0-0-0    304m    586Mi
clickhouse-keeper-keeper-0-0   20m     125Mi
```
ClickHouse Server + Keeper total: **324m CPU, 711 MiB memory**.

**Traditional observability namespace:**
```text
fluent-bit-traditional-2pncc                                      2m    6Mi
fluent-bit-traditional-gwr68                                      2m    6Mi
loki-stack-0                                                      3m    96Mi
loki-storage-exporter-747469dbb-vbh5s                             1m    18Mi
otel-collector-traditional-opentelemetry-collector-7554845tr2qw   4m    44Mi
tempo-0                                                           3m    44Mi
victoria-metrics-victoria-metrics-single-server-0                 6m    160Mi
vmagent-victoria-metrics-agent-595f9665b4-6m5zm                   3m    39Mi
```
Listed traditional components total: **24m CPU, 413 MiB memory**.

**Monitoring namespace:**
```text
kube-state-metrics-7cfd7f4c58-2rttv                      2m    30Mi
node-exporter-prometheus-node-exporter-hqqq4             4m    15Mi
node-exporter-prometheus-node-exporter-lpjrk             6m    16Mi
node-exporter-prometheus-node-exporter-mgmjv             6m    16Mi
otel-collector-opentelemetry-collector-59bd9c8d5-4khv7   8m    140Mi
```
Listed monitoring components total: **26m CPU, 217 MiB memory**.

The ClickHouse Server + Keeper snapshot (711 MiB, 324m) and the traditional namespace snapshot (413 MiB, 24m) were captured around the same time, but these are not yet a fully normalized resource comparison. The ClickHouse-side `monitoring` namespace contains the ClickHouse OTel Collector and cluster metrics collectors; decide whether to include these and which shared collectors should be excluded before calculating a final stack total. The Grafana memory panel screenshot shortly before this terminal output showed 733 MiB for ClickHouse + Keeper and 397 MiB for Traditional Stack, illustrating that the panel and `kubectl top` snapshots can differ due to time/aggregation/collection details. Keep both observations, do not overwrite one with the other.



### CPU panel query review — 2026-10-13

Reviewed Grafana panel: `ClickHouse vs Traditional — CPU Usage`.

**Traditional Stack query (VictoriaMetrics / PromQL):**
```promql
sum(
  rate(container_cpu_usage_seconds_total{
    namespace="traditional-observability",
    container!="",
    image!=""
  }[5m])
)
```

This applies PromQL `rate()` over a five-minute range and sums matching container CPU counters in the `traditional-observability` namespace.

**ClickHouse query logic:**
- Reads `container_cpu_usage_seconds_total` from `observability.otel_metrics_sum`.
- Filters to `Attributes['namespace'] = 'clickhouse'`, `Attributes['cpu'] = 'total'`, non-empty container/image, and the last five minutes.
- Groups by namespace, pod, and container.
- Calculates `(last counter value - first counter value) / elapsed seconds` using `argMax`/`argMin`; if the counter appears to decrease or elapsed time is not positive, returns zero.
- Sums the per-container estimates.

**Finding:** These are not mathematically equivalent rate calculations. PromQL `rate()` accounts for counter resets and range-vector behavior, while the ClickHouse SQL uses a first/last-sample estimate and returns zero on a decrease. The namespace scopes also differ (`clickhouse` vs `traditional-observability`). Treat current CPU panel values as indicative, not a fully normalized benchmark.

**Next validation:** Before replacing the existing query, inspect the actual stored metric schema and sample timestamps/counter values in ClickHouse, then determine whether an equivalent rate calculation can be implemented reliably with the available samples. Also decide whether supporting collectors should be counted in both stack totals or excluded from both. Do not change the dashboard query until the candidate query has been tested against terminal-derived data.



### ClickHouse CPU counter sample verification — 2026-10-13

Command run:
```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT Attributes['pod'] AS pod, Attributes['container'] AS container, count() AS samples, min(TimeUnix) AS first_sample, max(TimeUnix) AS last_sample, argMin(Value, TimeUnix) AS first_value, argMax(Value, TimeUnix) AS last_value FROM observability.otel_metrics_sum WHERE MetricName = 'container_cpu_usage_seconds_total' AND Attributes['namespace'] = 'clickhouse' AND Attributes['cpu'] = 'total' AND Attributes['container'] != '' AND Attributes['image'] != '' AND TimeUnix >= now() - INTERVAL 5 MINUTE GROUP BY pod, container ORDER BY pod, container"
```

Observed output:
```text
pod                          container         samples  first_sample         last_sample          first_value   last_value
clickhouse-clickhouse-0-0-0   clickhouse-server 10       2026-10-13 07:18:05  2026-10-13 07:22:32  1541.239857   1606.214675
clickhouse-keeper-keeper-0-0  clickhouse-keeper 10       2026-10-13 07:18:15  2026-10-13 07:22:44  175.136792    180.355221
```

Both containers have 10 samples over roughly 4 minutes 30 seconds. The counters increased in these first/last samples:
- ClickHouse Server: delta = `1606.214675 - 1541.239857` = **64.974818 CPU-seconds** over 267 seconds, estimated average **0.2433 CPU cores**.
- ClickHouse Keeper: delta = `180.355221 - 175.136792` = **5.218429 CPU-seconds** over 269 seconds, estimated average **0.0194 CPU cores**.
- Combined first/last estimate: approximately **0.2627 CPU cores**.

This estimate uses the displayed first/last samples and elapsed wall-clock time; it is not identical to PromQL `rate()` and may differ from Grafana due to sample timestamps, collection delay, range selection, and query evaluation time. This command does not establish that no counter reset occurred between samples; it only shows the first and last values are increasing. A reset inside the interval cannot be ruled out from this output alone.



### VictoriaMetrics in-pod API check — connection refused — 2026-10-13

Command attempted:
```bash
kubectl exec -n traditional-observability victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -qO- 'http://localhost:8428/api/v1/query' \
  --post-data='query=container_cpu_usage_seconds_total{namespace="traditional-observability",container!="",image!=""}' \
  --header='Content-Type: application/x-www-form-urlencoded'
```

Observed result:
```text
wget: can't connect to remote host: Connection refused
command terminated with exit code 1
```

Interpretation: the request could not connect to `localhost:8428` from inside that pod. This alone does not prove VictoriaMetrics is unhealthy; the actual listener address/port and pod/service configuration need to be checked. Next diagnostics:
```bash
kubectl get pods -n traditional-observability -o wide
kubectl get svc -n traditional-observability
```
Do not change the Grafana CPU query until the correct VictoriaMetrics API endpoint is confirmed.



### VictoriaMetrics pod/service inspection — 2026-10-13

Commands:
```bash
kubectl get pods -n traditional-observability -o wide
kubectl get svc -n traditional-observability
```

Relevant observed facts:
- VictoriaMetrics pod `victoria-metrics-victoria-metrics-single-server-0` is `1/1 Running`, pod IP `10.244.1.2`, node `kind-clickhouse-poc-worker2`.
- Headless service `victoria-metrics-victoria-metrics-single-server` exposes `8428/TCP`.
- NodePort service `vm-external` exposes `8428:31720/TCP`.
- The pod has 12 restarts (last restart reported 174 minutes ago); this is worth noting but does not by itself establish a current fault.

The earlier request to `http://localhost:8428/api/v1/query` from inside the pod returned `Connection refused`, so the listener/endpoint still needs verification. Next check:
```bash
kubectl exec -n traditional-observability victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -qO- 'http://victoria-metrics-victoria-metrics-single-server:8428/health'
```
If this fails, inspect the pod arguments and logs before changing any Grafana queries.



### VictoriaMetrics health and startup logs verified — 2026-10-13

The in-pod service-DNS health check succeeded:
```text
kubectl exec -n traditional-observability victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -qO- 'http://victoria-metrics-victoria-metrics-single-server:8428/health'
OK
```

The pod description confirms:
- Pod is `Running`, container `Ready: True`.
- Container argument includes `--httpListenAddr=:8428`.
- Readiness probe uses `/health`; liveness probe uses TCP port `http` (8428).
- Container has restarted 12 times; last terminated state is `Unknown`, exit code 255. The provided logs do not establish the cause of that restart.
- Startup log states `started server at http://0.0.0.0:8428/`.

**Interpretation:** VictoriaMetrics health endpoint works through the Kubernetes service DNS. The earlier request to `localhost:8428` returning `Connection refused` remains unexplained by the current evidence; do not infer that VictoriaMetrics is currently down.

Next, test the instant-query API using the working service DNS, preferably with an encoded query:
```bash
kubectl exec -n traditional-observability victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -qO- 'http://victoria-metrics-victoria-metrics-single-server:8428/api/v1/query?query=container_cpu_usage_seconds_total%7Bnamespace%3D%22traditional-observability%22%2Ccontainer!%3D%22%22%2Cimage!%3D%22%22%7D'
```
This checks whether the API returns matching counter series. It does not yet compare rates or establish parity with ClickHouse.



### VictoriaMetrics CPU counter API query verified — 2026-10-13

The instant-query API succeeded through the Kubernetes service DNS. Response:
- `status`: `success`
- `resultType`: `vector`
- `stats.seriesFetched`: `8`
- `stats.executionTimeMsec`: `1`

The query returned eight `container_cpu_usage_seconds_total` series in `traditional-observability`, including Fluent Bit (2 pods), Loki, Loki storage exporter, traditional OTel Collector, Tempo, VictoriaMetrics (`vmsingle`), and vmagent. The returned counter values were current samples at timestamp `1791876375` (Unix time), not CPU usage in cores.

**Interpretation:** VictoriaMetrics API and counter data are working through the service DNS. The returned counter values must not be compared directly with `kubectl top` CPU values; calculate a rate over a time window (e.g., PromQL `rate(...[5m])`) to obtain CPU cores. This API check confirms metric availability, not yet parity with the ClickHouse calculation.



### Traditional Stack CPU rate verified through VictoriaMetrics API — 2026-10-13

Executed the same PromQL expression used by the Grafana Traditional Stack CPU panel:
```promql
sum(
  rate(container_cpu_usage_seconds_total{
    namespace="traditional-observability",
    container!="",
    image!=""
  }[5m])
)
```

API response:
```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [1791876419, "0.0221089"]
      }
    ]
  },
  "stats": {
    "seriesFetched": "8",
    "executionTimeMsec": 0
  }
}
```

Result at evaluation time `1791876419`: **0.0221089 CPU cores**, or approximately **22.11 millicores**. The query fetched 8 matching series and executed in 0 ms as reported by the API. This is an API-side point-in-time evaluation of the same PromQL expression, not a simultaneous measurement with the ClickHouse query. Compare only after running both sides close together and documenting their different rate-calculation methods and scopes.



### ClickHouse CPU counter sample re-check — 2026-10-13

Command executed:
```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- \
clickhouse-client --query "
SELECT
    Attributes['pod'] AS pod,
    Attributes['container'] AS container,
    count() AS samples,
    min(TimeUnix) AS first_sample,
    max(TimeUnix) AS last_sample,
    argMin(Value, TimeUnix) AS first_value,
    argMax(Value, TimeUnix) AS last_value
FROM observability.otel_metrics_sum
WHERE MetricName = 'container_cpu_usage_seconds_total'
  AND Attributes['namespace'] = 'clickhouse'
  AND Attributes['cpu'] = 'total'
  AND Attributes['container'] != ''
  AND Attributes['image'] != ''
  AND TimeUnix >= now() - INTERVAL 5 MINUTE
GROUP BY pod, container
ORDER BY pod, container
"
```

Output:
```text
clickhouse-clickhouse-0-0-0  clickhouse-server  10  2026-10-13 07:23:26  2026-10-13 07:28:04  1619.114732  1681.064824
clickhouse-keeper-keeper-0-0 clickhouse-keeper  10  2026-10-13 07:23:50  2026-10-13 07:28:18  181.554329   186.558299
```

First/last sample estimates:
- ClickHouse Server: `(1681.064824 - 1619.114732) / 278 seconds` ≈ **0.22284 CPU cores**.
- ClickHouse Keeper: `(186.558299 - 181.554329) / 268 seconds` ≈ **0.01867 CPU cores**.
- Combined first/last estimate: **0.24151 CPU cores** (about 241.51 millicores).

These are estimates from the first and last stored samples per container, not a direct reproduction of Prometheus/VictoriaMetrics `rate()` semantics. Sample intervals are also not identical, and counters may reset between samples. Do not treat this as a definitive apples-to-apples result against the Traditional Stack API value of **0.0221089 CPU cores**. The current query scopes ClickHouse to the `clickhouse` namespace's server and Keeper containers; the Traditional query includes non-empty containers with images in `traditional-observability`, including that stack's other services. Normalize scope and take near-simultaneous readings before drawing a conclusion.



### ClickHouse CPU estimate from 10-minute sample window — 2026-10-13

Command executed against `observability.otel_metrics_sum` for `container_cpu_usage_seconds_total`, namespace `clickhouse`, CPU label `total`, and non-empty container/image attributes. The query calculated first/last counter deltas divided by elapsed seconds per container.

Output:
```text
clickhouse-clickhouse-0-0-0  clickhouse-server  20  565  1599.900629  1728.123196  0.226943
clickhouse-keeper-keeper-0-0  clickhouse-keeper  19  528  179.697799   189.692692   0.018930
```

Estimated CPU:
- ClickHouse Server: **0.226943 cores**
- ClickHouse Keeper: **0.018930 cores**
- Combined: **0.245873 cores** (approximately **245.87 millicores**)

The samples cover slightly different elapsed intervals (565 and 528 seconds), and the query uses first/last deltas rather than Prometheus/VictoriaMetrics `rate()` semantics. This is a useful estimate for validating the current Grafana value (~0.241 cores), but not an exact apples-to-apples comparison with the Traditional Stack query. Current Traditional Stack API result was **0.0221089 cores** at its evaluation timestamp; do not interpret the ratio as a final efficiency result until scope and measurement method are normalized.



### Log ingestion-rate bucket verification — 2026-10-13

The ClickHouse Grafana panel currently groups `observability.kubernetes_logs` into 30-second buckets and calculates `count() / 30.0` for logs/second. The Loki panel uses `sum by (namespace) (rate({namespace="clickhouse"}[1m]))`, which uses a 1-minute rate window; therefore, the two displayed series do not use identical calculation windows.

Terminal command:
```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- \
clickhouse-client --query "
SELECT
    toStartOfInterval(timestamp, INTERVAL 30 SECOND) AS time,
    count() AS log_count,
    round(count() / 30.0, 3) AS logs_per_second
FROM observability.kubernetes_logs
WHERE namespace = 'clickhouse'
  AND timestamp >= now() - INTERVAL 5 MINUTE
  AND timestamp <= now()
GROUP BY time
ORDER BY time
FORMAT Pretty
"
```

Output:
```text
2026-10-13 07:33:00  3  0.100
2026-10-13 07:33:30  6  0.200
2026-10-13 07:34:00  6  0.200
2026-10-13 07:34:30  6  0.200
2026-10-13 07:35:00  6  0.200
2026-10-13 07:35:30  6  0.200
2026-10-13 07:36:00  7  0.233
2026-10-13 07:36:30  6  0.200
2026-10-13 07:37:00  6  0.200
2026-10-13 07:37:30  6  0.200
2026-10-13 07:38:00  3  0.100
```

This confirms ClickHouse has log data throughout the displayed five-minute window; there are no empty 30-second buckets in this terminal result. The observed ClickHouse rate is mostly **0.200 logs/second** (6 logs/30 seconds), ranging from **0.100 to 0.233 logs/second**. The earlier missing/short green segment in Grafana cannot be explained by empty ClickHouse buckets alone; it may relate to query timestamps, query alignment, refresh timing, or visualization behavior and requires further checking. Do not change the query solely based on this output.



### Grafana ingestion-rate panel — 30-second Loki step verification — 2026-10-13

In Grafana panel `ClickHouse vs Loki — Log Ingestion Rate`, the Loki query remains:
```promql
sum by (namespace) (
  rate({namespace="clickhouse"}[1m])
)
```

The Loki query option `Type` is `Range`; its `Step` was changed from `auto` to `30s`. After running the query and returning to chart view, both ClickHouse (green) and Loki (yellow) series were visible, with timestamps aligned at approximately 30-second intervals.

The ClickHouse query still groups logs into fixed 30-second buckets:
```sql
SELECT
    toStartOfInterval(timestamp, INTERVAL 30 SECOND) AS time,
    count() / 30.0 AS logs_per_second
FROM observability.kubernetes_logs
WHERE namespace = 'clickhouse'
  AND timestamp >= now() - INTERVAL 5 MINUTE
  AND timestamp <= now()
GROUP BY time
ORDER BY time
```

This change improves evaluation-point alignment and chart readability. It does **not** make the calculations identical: ClickHouse calculates count-per-30-second-bucket, while Loki's `rate(...[1m])` estimates a per-second rate over a one-minute range. The chart now visually compares the series more clearly, but any quantitative conclusion should retain this methodology caveat.

---

## 12. Grafana: ClickHouse vs Loki — Log Ingestion Trend (2026-10-13)

### Panel configuration

- **Dashboard:** `ClickHouse POC`
- **Panel:** `ClickHouse vs Loki — Log Ingestion Trend`
- **Visualization:** Time series
- **Dashboard refresh:** 30 seconds
- **Loki query type:** Range
- **Loki step:** `60s` (`1m`)

**ClickHouse SQL currently used:**

```sql
SELECT
    toStartOfInterval(timestamp, INTERVAL 1 MINUTE) AS time,
    count() AS log_count
FROM observability.kubernetes_logs
WHERE namespace = 'clickhouse'
  AND timestamp >= now() - INTERVAL 5 MINUTE
  AND timestamp <= now()
GROUP BY time
ORDER BY time
```

**Loki LogQL currently used:**

```logql
sum by (namespace) (
  count_over_time({namespace="clickhouse"}[1m])
)
```

The Loki Step was changed from `auto` to `60s`. This aligns evaluation points to one-minute intervals, but it does **not** make Loki's rolling one-minute windows mathematically equivalent to ClickHouse's fixed minute buckets.

### Description to use in Grafana

> Compares the log ingestion trend between ClickHouse and Loki for the clickhouse namespace. ClickHouse counts logs in fixed one-minute event-time buckets, while Loki calculates a rolling one-minute count evaluated at 60-second intervals. Differences may occur due to window boundaries, partial intervals, and ingestion delays. This panel provides an indicative trend comparison and should not be used alone to verify exact log counts or log loss.

### Terminal verification — 2026-10-13

ClickHouse server timezone check:

```sql
SELECT
    timezone() AS server_timezone,
    now() AS server_now,
    toDateTime(1791882540) AS converted_start,
    toTimeZone(toDateTime(1791882540), 'UTC') AS start_utc
```

Observed output:

```text
UTC    2026-10-13 09:15:48    2026-10-13 09:09:00    2026-10-13 09:09:00
```

This confirmed that the ClickHouse server timezone was UTC and the test timestamp converted to `09:09:00 UTC`.

ClickHouse fixed minute-bucket counts for `[09:09:00, 09:15:00) UTC`:

| UTC minute | ClickHouse log count | First event | Last event |
|---|---:|---|---|
| 09:09 | 12 | 09:09:04 | 09:09:59 |
| 09:10 | 13 | 09:10:04 | 09:10:59 |
| 09:11 | 12 | 09:11:04 | 09:11:59 |
| 09:12 | 13 | 09:12:04 | 09:12:59 |
| 09:13 | 12 | 09:13:04 | 09:13:59 |
| 09:14 | 13 | 09:14:04 | 09:14:59 |

Loki range query sampled at 60-second steps returned rolling-window counts:

| Evaluation time (UTC) | Loki rolling 1-minute count |
|---|---:|
| 09:09 | 12 |
| 09:10 | 12 |
| 09:11 | 13 |
| 09:12 | 12 |
| 09:13 | 13 |
| 09:14 | 12 |
| 09:15 | 13 |

A Loki instant query at `09:10:00 UTC` returned `12` for `sum(count_over_time({namespace="clickhouse"}[1m]))`. ClickHouse's fixed `09:10` bucket had 13 events. This one-count difference is **not proof of log loss**, because the window definitions differ.

A raw Loki query returned sample entries from pod `clickhouse-keeper-keeper-0-0`, with raw event timestamps such as `2026-10-13T09:09:04.121272378Z`. The request used `limit=10`, so those 10 entries were only a sample, not the total number of logs.

### Interpretation and status

- The ClickHouse timezone and example UTC conversion were verified.
- Both systems returned log counts for the selected `clickhouse` namespace and interval.
- Loki Step `60s` is configured, and both series were visible in the Grafana panel screenshot.
- Counts were close but did not match at every evaluation point.
- ClickHouse uses fixed event-time buckets; Loki's `count_over_time(...[1m])` is a rolling window evaluated at each step.
- Partial current intervals and ingestion delay can also affect the latest point.
- **Status: indicative trend panel configured; exact count equivalence and log-loss validation are not established.** Do not declare an ingestion winner from this panel alone.


## Current Comparison Panel Status

The comparison dashboard work recorded so far covers:
- Log Count Comparison
- Log Ingestion Trend
- Log Ingestion Rate
- Storage Usage Comparison
- CPU Usage Comparison
- Memory Usage Comparison

The individual ClickHouse-only and Loki-only five-minute log-count panels were removed and are intentionally excluded from the active comparison panel list.

Query Latency Comparison has now been configured and both ClickHouse and Loki series were visible on 2026-10-14; its values remain indicative because the timing methods differ. The following comparison panels have been discussed but are not recorded as completed: Error Log Count Comparison, Data Freshness Comparison, Ingestion Health Comparison, Metrics Ingestion Comparison, and Trace Availability Comparison. Add these only after verifying that both sides have comparable data and query semantics.

## 13. Grafana: Query Latency Comparison — ClickHouse vs Loki (2026-10-14)

### Panel configuration and current observed values

- **Dashboard:** `ClickHouse POC`
- **Panel title:** `Query Latency Comparison (ClickHouse vs Loki)`
- **Visualization:** Bar gauge
- **Query A datasource:** `grafana-clickhouse-datasource-1`
- **Query B datasource:** `VM-1` (VictoriaMetrics, querying Loki's exported request metrics)
- **Legend A:** `ClickHouse`
- **Legend B:** `Loki`
- **Dashboard refresh:** 30 seconds

The Grafana panel showed the following values in the screenshot reviewed on 2026-10-14:

| Backend | Displayed latency | Measurement source |
|---|---:|---|
| ClickHouse | 3 ms | Average `query_duration_ms` from `system.query_log` for query IDs matching `latency_test_%` |
| Loki | 3.73 ms | Five-minute rate of Loki successful instant-query HTTP request duration sum divided by request count, converted to milliseconds |

These values are a point-in-time screenshot observation and may change as the Loki metric window and ClickHouse test query set change.

### ClickHouse query (Query A)

```sql
SELECT
    'ClickHouse' AS backend,
    avg(query_duration_ms) AS latency_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_id LIKE 'latency_test_%'
```

The query reports the average server-recorded duration for the matching test queries. It is based on the recorded `latency_test_%` query IDs; it is not a continuously refreshed benchmark of every dashboard query. Re-run the test queries and confirm the matching query-log records exist before interpreting the value as a fresh test result.

### Loki query (Query B, datasource `VM-1`)

```promql
rate(loki_request_duration_seconds_sum{route="loki_api_v1_query",status_code="200"}[5m])
/
rate(loki_request_duration_seconds_count{route="loki_api_v1_query",status_code="200"}[5m])
* 1000
```

This estimates the average successful Loki instant-query HTTP request duration over the previous five minutes and converts seconds to milliseconds. It is available only when matching request metrics have been scraped and requests occurred in the selected window. The value includes the timing scope represented by Loki's HTTP request-duration metric.

### Description configured in Grafana

> Compares query latency between ClickHouse and Loki for log-count queries over a five-minute window. ClickHouse latency is measured using `system.query_log`, while Loki latency is measured using HTTP response time. Since these measurements use different timing methods and query execution conditions, results are indicative and should not be treated as a strict apples-to-apples performance benchmark.

### Interpretation and status

- The panel now displays both named series: `ClickHouse` and `Loki`.
- The reviewed screenshot showed ClickHouse at **3 ms** and Loki at **3.73 ms**.
- The measurements are not strictly equivalent: ClickHouse uses a specific set of test query IDs, while Loki uses successful instant-query HTTP request metrics over a rolling five-minute window.
- The values do not prove that either backend is faster. For a fairer benchmark, execute equivalent count/filter queries against the same explicit time interval, measure both from the same client-side timing boundary, warm up both systems, collect repeated samples, and compare medians and spread.
- **Status: panel configured and both series visible; results are indicative, not a normalized benchmark.**
