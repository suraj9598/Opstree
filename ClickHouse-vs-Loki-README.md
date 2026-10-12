# ClickHouse vs Traditional Observability Stack

## Detailed POC README --- Explanation, Architecture, Testing, Results, and Limitations

> **Purpose:** This document is written so a DevOps engineer can explain
> the POC to a manager, interviewer, or teammate. It uses the actual
> setup and terminal observations collected during this POC.
> Measurements are observations from this Kind cluster, not a production
> benchmark.

------------------------------------------------------------------------

## 1. Executive Summary

This POC explores two ways to store and query observability data:

1.  **ClickHouse-centered approach:** ClickHouse is the main database
    for log data, metrics, and traces. ClickHouse Keeper is used for
    coordination. Fluent Bit sends Kubernetes logs directly to
    ClickHouse, while an OpenTelemetry Collector exports metrics and
    traces to ClickHouse.
2.  **Traditional multi-backend approach:** VictoriaMetrics stores
    metrics, Loki stores logs, and Tempo stores traces. `vmagent`
    scrapes metrics and sends them to VictoriaMetrics, Fluent Bit sends
    logs to Loki, and the OpenTelemetry Collector sends traces to Tempo.

Grafana is the visualization layer. It queries the relevant data source
and presents data in dashboards.

### What we verified

-   ClickHouse accepted a simple SQL query.
-   Loki readiness endpoint returned `ready`.
-   VictoriaMetrics returned HTTP `200 OK` and `OK` from its health
    endpoint.
-   Tempo readiness endpoint returned `ready`.
-   Recent five-minute log-count queries returned 65 records in both
    ClickHouse and Loki during near-time checks.
-   Recent ClickHouse query-log samples were in the low-millisecond
    range.
-   Loki reported low-millisecond query execution time, with queue time
    reported separately.
-   A short `kubectl top pods` snapshot showed the resource use of the
    two deployed component groups.
-   Test traces were found in ClickHouse and Tempo. A shared trace ID
    and matching identifying attributes were verified in both backends.

### What we have NOT proved

We have not proved that either architecture is definitively faster,
cheaper, or more storage-efficient. The paths do not have perfectly
identical pipelines, query timing fields have different meanings,
resource observations are snapshots, and the counters measure different
scopes. A fair performance verdict requires a controlled same-workload
test.

------------------------------------------------------------------------

## 2. What Is Observability?

Observability helps us understand what an application or infrastructure
system is doing by looking at three main kinds of telemetry:

  -----------------------------------------------------------------------
  Telemetry               What it tells us        Example
  ----------------------- ----------------------- -----------------------
  Logs                    What happened, as       A pod reports an error
                          individual events       

  Metrics                 How much/how often      CPU usage or request
                          something is happening, rate
                          as numeric time series  

  Traces                  How one request moves   Request enters service
                          through components      A, calls service B,
                                                  then a database
  -----------------------------------------------------------------------

A practical observability platform lets an engineer move from a symptom
to a cause. For example:

1.  A dashboard shows high latency.
2.  A metric shows which service or node is affected.
3.  Logs reveal an error message.
4.  A trace shows which operation took the time.

The three telemetry types complement one another; they are not
interchangeable.

------------------------------------------------------------------------

## 3. POC Environment

### Kubernetes and namespaces

The Kubernetes context used for this POC was:

``` bash
kubectl config current-context
```

Observed context: `kind-kind-clickhouse-poc`.

The main namespaces and responsibilities are:

  -----------------------------------------------------------------------------
  Namespace                     Components              Responsibility
  ----------------------------- ----------------------- -----------------------
  `clickhouse`                  ClickHouse, ClickHouse  ClickHouse database and
                                Keeper                  coordination

  `monitoring`                  ClickHouse-side         Collect/export
                                OpenTelemetry           telemetry for the
                                Collector, node         ClickHouse path
                                exporter,               
                                kube-state-metrics      

  `logging`                     ClickHouse-side Fluent  Read Kubernetes
                                Bit                     container logs and send
                                                        them to ClickHouse

  `traditional-observability`   Loki, VictoriaMetrics,  Traditional
                                vmagent, Tempo,         multi-backend path
                                traditional OTel        
                                Collector, Fluent Bit,  
                                Loki storage exporter   
  -----------------------------------------------------------------------------

Grafana runs separately from the Kind cluster.

### Important terminology

-   **Pod:** A Kubernetes unit that runs one or more containers.
-   **Namespace:** A logical grouping of Kubernetes resources.
-   **Datasource:** A configured connection that Grafana uses to query a
    backend.
-   **Exporter:** A component that writes telemetry to a destination.
-   **Scrape:** A metrics collector periodically requests metrics from
    an endpoint.
-   **Ingestion:** The process of accepting and storing incoming
    telemetry.
-   **Query latency:** The time taken to execute a query. Different
    tools may measure different parts of the request.
-   **Retention:** How long telemetry is kept before it is deleted.
-   **Cardinality:** The number of unique time series or label
    combinations. High cardinality can increase resource usage.

------------------------------------------------------------------------

## 4. Architecture A --- ClickHouse-Centered Observability

### Components

#### ClickHouse

ClickHouse is a column-oriented database designed for analytical
queries. In this POC, the `observability` database contains tables for
Kubernetes logs, OpenTelemetry metrics, and traces.

Useful tables observed include:

-   `observability.kubernetes_logs`
-   `observability.otel_metrics_gauge`
-   `observability.otel_metrics_sum`
-   `observability.otel_metrics_summary`
-   `observability.otel_metrics_histogram`
-   `observability.otel_traces`

The `kubernetes_logs` table stores the Kubernetes logs used in our
log-count comparison.

#### ClickHouse Keeper

Keeper provides coordination for ClickHouse deployments that need it,
including replicated/distributed setups. In this POC it runs as a
separate pod. It is not the Grafana datasource; Grafana queries
ClickHouse.

#### Fluent Bit

The ClickHouse-side Fluent Bit DaemonSet reads files matching
`/var/log/containers/*.log`, enriches records with Kubernetes metadata,
and sends them to ClickHouse over HTTP port `8123`.

The output targets:

``` text
clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:8123
```

It inserts into:

``` text
observability.kubernetes_logs
```

The configured output uses `JSONEachRow`/`json_lines` and adds a
timestamp field.

#### OpenTelemetry Collector

The ClickHouse-side Collector receives telemetry and exports selected
pipelines to ClickHouse.

Observed configuration details: - ClickHouse exporter has
`async_insert: true` and `create_schema: true`. - Metrics pipeline
exports to ClickHouse. - Traces pipeline exports to ClickHouse. - Logs
pipeline uses a `debug` exporter only.

**Important:** The Kubernetes logs being compared in `kubernetes_logs`
come through Fluent Bit. The Collector's logs pipeline is not the path
that writes these records into ClickHouse.

### ClickHouse data path

``` text
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

Metrics endpoints --> OTel Collector --> ClickHouse
Application traces --> OTel Collector --> ClickHouse
                                      |
                                      v
                                   Grafana
```

This diagram is simplified. The actual telemetry routes depend on the
receiver, pipeline, and exporter configurations.

------------------------------------------------------------------------

## 5. Architecture B --- Traditional Multi-Backend Observability

### VictoriaMetrics --- metrics backend

VictoriaMetrics stores metrics as time series. In this POC, `vmagent`
scrapes configured targets and writes samples to VictoriaMetrics.

Observed scrape jobs included: - node exporter - kube-state-metrics -
cAdvisor through the Kubernetes node proxy - Loki metrics -
VictoriaMetrics self-metrics - Fluent Bit metrics - Loki storage
exporter

The configured global scrape interval was 30 seconds.

The VictoriaMetrics service used for in-cluster checks was:

``` text
victoria-metrics-victoria-metrics-single-server.traditional-observability.svc.cluster.local:8428
```

### Loki --- logs backend

Loki stores and queries logs. Fluent Bit sends logs to Loki and adds
labels such as `namespace`, `pod`, and `container`.

A LogQL query selects logs by labels and can count matching entries over
a time window.

### Tempo --- traces backend

Tempo stores distributed traces. The traditional OpenTelemetry Collector
sends traces to Tempo through OTLP gRPC at:

``` text
tempo.traditional-observability.svc.cluster.local:4317
```

Tempo exposes its query API on port `3200` in this setup.

### Traditional OTel Collector caveat

The observed traditional Collector configuration used: - traces pipeline
→ Tempo through OTLP gRPC - logs pipeline → debug exporter - metrics
pipeline → debug exporter

Metrics were sent to VictoriaMetrics by `vmagent`, and Kubernetes logs
were sent to Loki by Fluent Bit. Do not describe the traditional
Collector as the component that writes all logs and metrics into the
traditional backends.

### Traditional data path

``` text
Metrics endpoints --> vmagent --> VictoriaMetrics
Kubernetes logs --> Fluent Bit --> Loki
Application traces --> OTel Collector --> Tempo

Grafana --> VictoriaMetrics / Loki / Tempo
```

------------------------------------------------------------------------

## 6. Why Use Grafana?

Grafana is the visualization and exploration layer. It does not itself
store the telemetry from these backends. Instead, it queries each
configured datasource.

A dashboard can contain: - Stat panels for a single value, such as logs
in the last five minutes. - Time-series panels for metrics over time. -
Logs panels to inspect log lines. - Trace search panels to find and open
traces. - Text panels to explain test scope and limitations.

The POC has these datasource entries in Grafana:

  -----------------------------------------------------------------------------------
  Datasource name                     Type                    Purpose
  ----------------------------------- ----------------------- -----------------------
  `grafana-clickhouse-datasource-1`   ClickHouse              Query ClickHouse SQL
                                                              tables

  `loki-1`                            Loki                    Query logs with LogQL

  `VM-1`                              Prometheus-compatible   Query VictoriaMetrics
                                                              through its
                                                              Prometheus-compatible
                                                              API

  `tempo`                             Tempo                   Search and open traces
  -----------------------------------------------------------------------------------

The datasource list also showed other datasource entries, including a
second ClickHouse datasource and other Loki/Prometheus entries. Use the
four named entries above for the comparison panels, and confirm their
connection settings before building panels.

### Datasource networking warning

The screenshots showed URLs such as `127.0.0.1:8123`, `127.0.0.1:32446`,
`127.0.0.1:31720`, and `127.0.0.1:31000`. `127.0.0.1` means "this same
network namespace." If Grafana runs on another server, its own loopback
is not the laptop's loopback or a Kind pod's loopback. A URL can work
only if the service is exposed or forwarded in a way reachable from the
Grafana server. If a panel returns an error or no data, check network
reachability and the datasource's **Save & test** result first.

------------------------------------------------------------------------

## 7. Health Checks and What They Mean

These commands were run as final basic health checks.

### ClickHouse

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client --query "SELECT 1"
```

Observed output:

``` text
1
```

Meaning: the client connected to ClickHouse and the server executed a
simple SQL query.

### Loki

``` bash
kubectl exec -n traditional-observability loki-stack-0 -- \
  wget -qO- http://localhost:3100/ready
```

Observed output:

``` text
ready
```

Meaning: Loki's readiness endpoint reported ready.

### VictoriaMetrics

``` bash
kubectl exec -n traditional-observability \
  victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -S -O- http://victoria-metrics-victoria-metrics-single-server:8428/health
```

Observed HTTP status: `HTTP/1.1 200 OK`; body: `OK`.

Meaning: the service endpoint responded successfully to its health
check.

### Tempo

``` bash
kubectl exec -n traditional-observability tempo-0 -- \
  wget -qO- http://localhost:3200/ready
```

Observed output:

``` text
ready
```

Meaning: Tempo's readiness endpoint reported ready.

### What these checks do not prove

A healthy endpoint does not by itself prove that every telemetry type is
being ingested correctly, that data is complete, or that a Grafana query
works. Those need separate ingestion and query checks.

------------------------------------------------------------------------

## 8. Log Count Comparison

The goal was to compare the number of log entries in the `clickhouse`
namespace over a rolling five-minute window.

### ClickHouse SQL

``` sql
SELECT
    now() AS checked_at,
    now() - INTERVAL 5 MINUTE AS window_start,
    count() AS log_count
FROM observability.kubernetes_logs
WHERE namespace = 'clickhouse'
  AND timestamp >= now() - INTERVAL 5 MINUTE
  AND timestamp <= now()
FORMAT PrettyCompact
```

One observed result at `2026-10-12 07:29:50`:

  Checked at   Window start     Count
  ------------ -------------- -------
  07:29:50     07:24:50            65

### Loki LogQL query

URL-encoded query used:

``` text
sum(count_over_time({namespace="clickhouse"}[5m]))
```

Command:

``` bash
kubectl exec -n traditional-observability loki-stack-0 -- \
  sh -c 'wget -qO- "http://localhost:3100/loki/api/v1/query?query=sum%28count_over_time%28%7Bnamespace%3D%22clickhouse%22%7D%5B5m%5D%29%29"'
```

A near-time result returned: - Count: `65` - `execTime`: `0.002746459`
seconds, about `2.75 ms` - `queueTime`: `0.006749964` seconds, about
`6.75 ms`

Three immediately repeated Loki requests earlier returned count `66`
each, with `execTime` values of about `5.11 ms`, `4.44 ms`, and
`4.51 ms`.

### What can we conclude?

The near-time checks returned 65 in both systems. That is useful
evidence that both paths had a similar number of matching records at
that point.

However, **equal counts do not prove that the underlying records are
identical**. The queries were run one after another rather than at
precisely the same instant, and the five-minute window moves
continuously. To prove record-level equality, the test needs stable
unique IDs or a controlled test batch and a comparison of those IDs.

------------------------------------------------------------------------

## 9. Query Latency Results

### ClickHouse server-side query log

The query history was read from `system.query_log`:

``` sql
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
FORMAT PrettyCompact
```

Recent samples:

  Event time     Query duration   Rows read   Bytes read
  ------------ ---------------- ----------- ------------
  07:28:24                 9 ms       8,443      231,749
  07:28:23                 5 ms       8,437      231,542
  07:28:08                 5 ms       8,348      229,024

Mean of these three samples: approximately **6.3 ms**.

Another query at `07:29:50` was recorded at 15 ms, reading 8,836 rows
and 242,799 bytes. This shows that individual query times vary.

### Loki timings

Loki's response exposes `stats.summary.execTime` and `queueTime`. The
three repeated execution times were: - 5.108 ms - 4.436 ms - 4.513 ms

Mean `execTime`: approximately **4.69 ms**.

One later response reported: - `execTime`: 2.75 ms - `queueTime`: 6.75
ms

### Why we should not announce a latency winner yet

-   ClickHouse's `query_duration_ms` is a server-side query-log field.
-   Loki's `execTime` is a field in Loki's query statistics, and Loki
    also reports queue time separately.
-   The requests were not a formal benchmark with a fixed dataset, fixed
    query window, warm/cold cache control, or many repeated trials.
-   Network/client overhead is not represented identically.

**Safe explanation:** "Both backends returned the count query in
low-millisecond server-reported execution times during this POC. The
measurements are encouraging but are not directly equivalent enough to
declare a winner."

------------------------------------------------------------------------

## 10. Resource Utilization Snapshot

Metrics Server was installed so `kubectl top pods` could report CPU and
memory. A local Kind kubelet certificate issue was worked around by
enabling `--kubelet-insecure-tls` for Metrics Server. This is acceptable
only as a local POC workaround; do not copy that insecure setting into
production.

Command:

``` bash
kubectl top pods -n clickhouse
kubectl top pods -n traditional-observability
```

Observed snapshot:

  Pod/component                          CPU        Memory
  ------------------------------- ---------- -------------
  ClickHouse                            113m       716 MiB
  ClickHouse Keeper                      18m       125 MiB
  **ClickHouse + Keeper total**     **131m**   **841 MiB**
  Traditional Fluent Bit pod 1            2m         7 MiB
  Traditional Fluent Bit pod 2            1m         9 MiB
  Loki                                    6m        87 MiB
  Loki storage exporter                   2m        16 MiB
  Traditional OTel Collector              3m        76 MiB
  Tempo                                   3m       113 MiB
  VictoriaMetrics                         7m       144 MiB
  vmagent                                 3m        29 MiB
  **Traditional stack total**        **24m**   **481 MiB**

Note: Adding the displayed traditional CPU values gives 26m in this
particular table (2+1+6+2+3+3+7+3 = 27m); pod metrics are rounded to
whole millicores, and the displayed total should be treated as
approximate. The previously captured snapshot's displayed components sum
to around 27m CPU. Memory total is approximately 481 MiB.

### Interpretation

In this one snapshot, ClickHouse plus Keeper used more CPU and memory
than the listed traditional stack pods. But this is **not an
apples-to-apples efficiency conclusion** because: - the two
architectures have different component responsibilities and counts; -
the workload and telemetry rates were not controlled; - it is one short
sample, not a time average or peak measurement; - ClickHouse was
carrying substantial accumulated data in its tables.

For a fair comparison, record CPU and memory repeatedly over the same
test interval and the same incoming workload. Include limits/requests,
restart counts, and ingestion completeness.

------------------------------------------------------------------------

## 11. Ingestion Counter Snapshots

### ClickHouse process-wide insert counters

Command:

``` sql
SELECT
    event,
    value
FROM system.events
WHERE event IN ('InsertQuery', 'InsertedRows', 'InsertedBytes')
ORDER BY event
FORMAT PrettyCompact
```

First observed snapshot: - `InsertQuery`: 24,082 - `InsertedRows`:
14,026,999 - `InsertedBytes`: 6,418,256,742

Second observed snapshot: - `InsertQuery`: 24,167 - `InsertedRows`:
14,075,102 - `InsertedBytes`: 6,440,548,892

Differences: - 85 insert queries - 48,103 rows - 22,292,150 bytes

If the interval was approximately 60 seconds, the approximate rates
are: - 1.42 insert queries/second - 802 rows/second - 371.5 KB/second
(decimal bytes)

**Scope warning:** These ClickHouse counters are process-wide. They are
not guaranteed to represent only Kubernetes logs or only this POC's one
table.

### VictoriaMetrics storage counter

Query:

``` bash
kubectl exec -n traditional-observability \
  victoria-metrics-victoria-metrics-single-server-0 -- \
  wget -qO- \
  'http://victoria-metrics-victoria-metrics-single-server:8428/api/v1/query?query=vm_rows_added_to_storage_total'
```

Observed samples: - At Unix timestamp `1791790429`: `9,655,167` - At
Unix timestamp `1791790514`: `9,704,965`

Difference: 49,798 rows over 85 seconds, or approximately 586
rows/second.

### Why these rates cannot be directly compared

ClickHouse `InsertedRows` and VictoriaMetrics
`vm_rows_added_to_storage_total` have different definitions and scopes.
One counts rows inserted into ClickHouse process-wide; the other counts
rows added to VictoriaMetrics storage. The counters may include
different sources and types of telemetry. These numbers are useful as
observations, not as proof that one backend has higher ingestion
throughput.

------------------------------------------------------------------------

## 12. Storage Measurements

A ClickHouse `system.parts` snapshot reported approximately: -
`otel_metrics_gauge`: 101.0 million rows, about 1.02 GiB -
`otel_metrics_sum`: 58.3 million rows, about 652 MiB -
`kubernetes_logs`: 3.76 million rows, around 62--65 MB in the captured
snapshots - `otel_metrics_summary`: around 49.8 thousand rows, about 1.3
MB - `otel_metrics_histogram`: around 12.3 thousand rows, under 0.5 MB -
`otel_traces`: 3 rows, about 19 KiB - `otel_traces_trace_id_ts`: 3 rows,
about 3.8 KiB

Observed directory usage: - ClickHouse `/var/lib/clickhouse`: about 3.1
GB - VictoriaMetrics `/storage`: around 98.6 MB initially - Loki
`/data`: around 82.8 MB - Tempo `/var/tempo`: around 76 KB at the time
measured

PVC requests: - ClickHouse: 5 Gi - ClickHouse Keeper: 2 Gi -
VictoriaMetrics: 16 Gi - Loki: 5 Gi - Tempo: 5 Gi

### Storage interpretation

A PVC's requested capacity is not the same as its used storage.
Directory sizes and ClickHouse part sizes also have different scopes.
Data volume, retention, compression, indexing, compaction, and workload
were not held constant, so these numbers do not prove a
storage-efficiency winner.

------------------------------------------------------------------------

## 13. Trace Verification

The POC sent test traces to both paths and checked whether the traces
could be queried back.

### ClickHouse trace

A fresh test trace was found in `observability.otel_traces`: - Span
name: `clickhouse-fresh-test` - Service name:
`clickhouse-trace-verification` - Attribute:
`test.run=fresh-verification`

### Tempo trace

Tempo's trace API and search API returned test traces, including a trace
named `tempo-ingestion-verification`.

### Shared trace ID check

The shared comparison trace ID was:

``` text
64126405d0a7d5af2f83dcf6660f3e71
```

It was found in both ClickHouse and Tempo with matching identifying
information: - Span name: `shared-comparison-span` - Service name:
`shared-trace-comparison` - Comparison attributes included
`comparison.id=shared-trace-001` and
`comparison.test=same-span-both-backends`

The span IDs differed, because the test generation methods were not
identical at the span level. Therefore the accurate claim is: **the same
trace ID and matching identifying attributes were found in both
backends**. Do not claim the exact same span record was proven.

------------------------------------------------------------------------

## 14. Reliability and Operational Considerations

A comparison should not look only at query latency. Operational
questions also matter:

-   How many services must be installed, upgraded, and backed up?
-   How are retention and deletion configured?
-   How do engineers troubleshoot missing data?
-   How are authentication, authorization, and network access secured?
-   What happens when a component or node fails?
-   Are dashboards and alerts easy to maintain?
-   Can logs, metrics, and traces be correlated reliably?
-   What are the backup, restore, and disaster recovery procedures?
-   What are the CPU/memory requests and limits?
-   What are the ingestion lag and dropped-record rates?

### ClickHouse-centered approach

Potential benefit: logs, metrics, and traces can be queried in one
database/query environment when schemas and ingestion are designed
appropriately.

Operational cost: ClickHouse schema, SQL, storage sizing, merges,
replication, and Keeper/cluster operations need to be understood and
maintained.

### Multi-backend approach

Potential benefit: separate systems can be selected and tuned for their
telemetry type, with familiar APIs and query languages.

Operational cost: multiple backends need their own capacity planning,
retention, upgrades, troubleshooting, and access configuration.
Cross-signal correlation and dashboard consistency require deliberate
configuration.

These are architectural trade-offs, not measured wins from this small
POC.

------------------------------------------------------------------------

## 15. Grafana Dashboard Plan

The dashboard should be built manually so each panel can be tested and
explained.

Suggested panel order:

1.  **ClickHouse Logs --- Last 5 Minutes** --- Stat panel using
    ClickHouse SQL.
2.  **Loki Logs --- Last 5 Minutes** --- Stat panel using LogQL.
3.  **ClickHouse Query Duration** --- Time series/table from
    `system.query_log`, with care to query the correct ClickHouse
    datasource.
4.  **Loki Query Execution Time** --- use Loki query statistics where
    the datasource supports it; otherwise keep measured values in the
    benchmark notes rather than pretending they are a continuously
    scraped metric.
5.  **VictoriaMetrics Targets Up** --- Prometheus-compatible query
    `sum(up)`.
6.  **VictoriaMetrics Rows Added/sec** ---
    `rate(vm_rows_added_to_storage_total[5m])`.
7.  **Traditional Stack Pod CPU** --- cAdvisor metric, if exposed by the
    configured scrape target.
8.  **Traditional Stack Pod Memory** --- cAdvisor metric, if exposed by
    the configured scrape target.
9.  **ClickHouse CPU and Memory** --- use a verified Kubernetes/cAdvisor
    or node-level metric, not a guessed metric name.
10. **Loki Logs Explorer** --- `{namespace="clickhouse"}`.
11. **Tempo Trace Search** --- search for a known service or trace ID.
12. **POC Notes / Limitations** --- text panel explaining the
    benchmark's scope.

Before adding a panel: 1. Select the intended datasource. 2. Test the
query in Explore or the panel editor. 3. Verify the result shape. 4.
Choose visualization and unit. 5. Give it a clear title. 6. Save the
dashboard. 7. Repeat with the next panel.

The dashboard should not compare values with different meanings on the
same axis. A historical benchmark table is not live data; label it as a
recorded snapshot.

------------------------------------------------------------------------

## 16. Controlled Benchmark Plan for a Stronger Conclusion

To compare the architectures fairly:

1.  Generate a repeatable test workload with known log, metric, and
    trace counts.
2.  Send the same data and equivalent labels/attributes to both paths.
3.  Use synchronized test start and end times.
4.  Use semantically equivalent queries and equivalent time windows.
5.  Run multiple trials and record p50/p95 latency, not just a single
    average.
6.  Separate client round-trip time, queue time, and backend execution
    time.
7.  Measure ingestion lag and verify no records were dropped.
8.  Compare records by stable unique ID; matching counts alone are
    insufficient.
9.  Sample CPU, memory, and disk I/O at regular intervals under the same
    workload.
10. Compare storage growth after equal data volume and equal retention.
11. Record component versions, configuration, resource limits, and test
    duration.
12. Repeat after warming up caches and also test cold-start behavior if
    relevant.

Only after this should the team make a defensible performance or cost
recommendation.

------------------------------------------------------------------------

## 17. Suggested Explanation for a Manager or Interview

You can explain the POC like this:

> "We created a local Kind Kubernetes POC to compare a
> ClickHouse-centered observability design with a traditional design
> using VictoriaMetrics for metrics, Loki for logs, and Tempo for
> traces. In the ClickHouse path, Fluent Bit sends Kubernetes logs to a
> ClickHouse table, and the OpenTelemetry Collector sends configured
> metrics and traces to ClickHouse. In the traditional path, Fluent Bit
> sends logs to Loki, vmagent scrapes metrics into VictoriaMetrics, and
> the OpenTelemetry Collector sends traces to Tempo. Grafana is the
> common visualization layer.
>
> We validated the health endpoints and tested log-count queries over a
> rolling five-minute window. A near-time check returned 65 logs in both
> ClickHouse and Loki. Recent query samples were in the low-millisecond
> range. We also collected one CPU/memory snapshot, storage
> observations, ingestion-counter snapshots, and verified test traces in
> both systems.
>
> These are preliminary POC observations. They are not a final
> performance verdict because the pipelines and counter definitions
> differ, the workload was not fully controlled, and the queries were
> not executed at exactly the same instant. The next step is to build
> Grafana panels manually and run a controlled benchmark with the same
> workload, synchronized windows, repeated latency trials, resource
> sampling, and record-level validation."

------------------------------------------------------------------------

## 18. Final Conclusion

The POC successfully demonstrated that the two telemetry paths are
running and can return selected logs, metrics, and traces. It also
captured useful baseline observations for query execution, resource
usage, ingestion counters, and storage.

The evidence supports saying: - Both paths are operational for the
tested functions. - Near-time five-minute log counts matched once at
65. - The sampled backend query execution times were low-millisecond. -
Resource and ingestion counters were captured for further analysis. -
Test traces were retrievable in both backends.

The evidence does **not** support saying that ClickHouse is universally
faster, cheaper, more resource-efficient, or more storage-efficient than
the multi-backend stack. That decision requires an apples-to-apples
benchmark.

**POC status:** Functional validation completed for the tested checks;
Grafana dashboard panels and controlled benchmark remain follow-up work.
