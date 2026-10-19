# ClickHouse vs Tempo: Distributed Tracing POC

## 1. Overview

This README documents a proof of concept (POC) comparing **ClickHouse**
and **Grafana Tempo** as distributed-tracing backends. It records the
Kubernetes architecture, controlled trace test, preliminary latency
measurements, ClickHouse storage, duplicate rows, memory-limit errors,
troubleshooting commands, and remaining validation.

**Status:** Initial comparison is complete, but this is not a
production-grade benchmark. The dataset is small, the ClickHouse and
Tempo requests were not fully equivalent, and Tempo's trace-only disk
usage has not been measured.

## 2. Goals

-   Send the same test trace to ClickHouse and Tempo.
-   Verify that each backend can retrieve the trace.
-   Record initial latency observations.
-   Inspect ClickHouse trace-table storage.
-   Identify duplicate span rows.
-   Investigate memory-limit errors during telemetry ingestion.
-   Document limitations and remaining work before drawing a production
    conclusion.

## 3. Environment

  ----------------------------------------------------------------------------------------------------------------------------
  Component         Namespace                     Resource / endpoint                                        Notes
  ----------------- ----------------------------- ---------------------------------------------------------- -----------------
  ClickHouse        `clickhouse`                  Pod `clickhouse-clickhouse-0-0-0`                          Version reported:
                                                                                                             `26.9.4`

  ClickHouse HTTP   `clickhouse`                  Service `clickhouse-http`, local port `8123`               Direct HTTP
                                                                                                             benchmark

  Tempo             `traditional-observability`   Service `tempo`, port `3200`                               Version
                                                                                                             previously
                                                                                                             observed: `2.9.0`

  ClickHouse OTel   `monitoring`                  Pod                                                        Exports telemetry
  Collector                                       `otel-collector-opentelemetry-collector-59bd9c8d5-4khv7`   to ClickHouse

  Tempo OTel        `traditional-observability`   Service                                                    Exports OTLP gRPC
  Collector                                       `otel-collector-traditional-opentelemetry-collector`       to Tempo

  Kubernetes        ---                           `kind-kind-clickhouse-poc`                                 Local Kind
  context                                                                                                    cluster
  ----------------------------------------------------------------------------------------------------------------------------

### Port-forwarding

Run these in separate terminals and keep them active during direct
requests:

``` bash
kubectl port-forward -n traditional-observability svc/tempo 3200:3200
```

``` bash
kubectl port-forward -n clickhouse svc/clickhouse-http 8123:8123
```

## 4. High-level architecture

``` text
                         Controlled OTLP test trace
                                    |
                   +----------------+----------------+
                   |                                 |
                   v                                 v
       ClickHouse OTel Collector             Tempo OTel Collector
             namespace: monitoring      namespace: traditional-observability
                   |                                 |
          ClickHouse exporter                 OTLP gRPC exporter
                   |                                 |
                   v                                 v
          ClickHouse service                       Tempo service
                   |                                 |
                   v                                 v
       observability.otel_traces               Tempo local backend
       observability.otel_logs                 /var/tempo
       observability.otel_metrics_*             WAL: /var/tempo/wal
```

The same test trace was sent independently to the two collectors. Both
ingestion requests returned HTTP `200`.

### ClickHouse path

1.  OTLP payload is sent to the ClickHouse-facing collector.
2.  The ClickHouse exporter writes telemetry into ClickHouse.
3.  Trace data is queryable in `observability.otel_traces`.
4.  This environment also stores logs, metrics, and ClickHouse system
    data.

### Tempo path

1.  OTLP payload is sent to the Tempo-facing collector.
2.  The collector exports OTLP over gRPC to Tempo on port `4317`.
3.  Tempo uses its local backend under `/var/tempo`; its WAL path is
    `/var/tempo/wal`.
4.  Trace retrieval uses the Tempo HTTP API on port `3200`.

**Storage caveat:** ClickHouse's PVC contains traces, logs, metrics, and
internal system data. Tempo's backend is trace-focused. Total ClickHouse
PVC usage must not be directly compared with Tempo's PVC capacity as a
trace-only storage comparison.

## 5. Controlled test trace

  Field             Value
  ----------------- ---------------------------------------------------------
  Trace ID          `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
  Span ID           `bbbbbbbbbbbbbbbb`
  Service name      `backend-comparison-test`
  Span name         `same-trace-clickhouse-test`
  Span kind         `INTERNAL`
  Start timestamp   `1792440000000000000` ns
  End timestamp     `1792440000100000000` ns
  Duration          `100000000` ns = **100 ms**
  Status            Unset / no explicit status value in returned Tempo span

### Retrieve from Tempo

With the Tempo port-forward active:

``` bash
curl -s http://localhost:3200/api/traces/aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

The response contained service `backend-comparison-test`, span
`same-trace-clickhouse-test`, and one span. Tempo encoded the span ID in
the API response as base64 (`u7u7u7u7u7s=`).

### Verify in ClickHouse

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT
    count() AS total_rows,
    uniqExact(SpanId) AS unique_spans
FROM observability.otel_traces
WHERE TraceId = 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa'
"
```

Observed output:

``` text
2    1
```

This means the trace ID matched two rows but only one unique span ID. A
full-row comparison across the trace table's listed columns also
returned two rows and one unique full-row tuple. The rows were identical
across the compared columns. The cause of the duplicate has not yet been
established.

## 6. ClickHouse trace-table storage

Command used:

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    sum(rows) AS rows
FROM system.parts
WHERE active
  AND database = 'observability'
  AND table = 'otel_traces'
GROUP BY table
FORMAT PrettyCompact
"
```

Observed result:

  Table             On-disk size   Rows
  --------------- -------------- ------
  `otel_traces`      `31.93 KiB`    `5`

This is the entire `observability.otel_traces` table, not only the
controlled test trace. Tempo's PVC capacity was 5 GiB, but actual
trace-only disk usage was not measured. A storage winner cannot be
declared from these measurements.

## 7. Initial latency benchmark

Five-run direct HTTP measurements were recorded for ClickHouse and
Tempo.

  -------------------------------------------------------------------------
  Request         Five-run                     Average               Median
                  observations                         
  --------------- --------------- -------------------- --------------------
  ClickHouse      14.764, 6.129,              7.798 ms             6.129 ms
  `count()` via   4.882, 7.909,                        
  HTTP            5.305 ms                             

  ClickHouse      19.613, 33.209,            17.109 ms            14.835 ms
  trace row       7.035, 14.835,                       
  retrieval       10.852 ms                            
  (`SELECT *`)                                         
  via HTTP                                             

  Tempo trace     41.522, 7.169,             15.064 ms             9.022 ms
  lookup API via  6.810, 9.022,                        
  HTTP            10.797 ms                            
  -------------------------------------------------------------------------

Earlier `kubectl exec` tests included Kubernetes exec and client startup
overhead. Those tests averaged approximately 0.17 seconds for ClickHouse
count and 0.088 seconds for Tempo trace retrieval; they are not directly
comparable because the operations and command overhead differed.

### Interpretation

-   ClickHouse `count()` is not the same operation as full trace
    retrieval.
-   ClickHouse `SELECT *` returns table rows for the trace ID.
-   Tempo's API returns a trace structure with span data.
-   ClickHouse returned two duplicate rows for this test trace; Tempo
    returned one span.
-   The result structure and amount of returned data differ.

These are preliminary observations, not proof that one backend is
faster. A fair test should use equivalent query intent, a larger
controlled dataset, known span counts, warm-up requests, repeated runs,
response-size measurements, and error-rate tracking.

## 8. ClickHouse memory and ingestion findings

The ClickHouse-facing OTel Collector repeatedly logged failures
exporting **metrics**. The ClickHouse error was code `241`, a
memory-limit error. Representative details:

``` text
(total) memory limit exceeded: would use 1.80 GiB
attempt to allocate chunk of about 4.17 MiB
OvercommitTracker decision: Query was selected to stop by OvercommitTracker
```

Failed exports included gauge, sum, histogram, and summary metrics,
followed by retries. Separately, ClickHouse `system.query_log` recorded
code `241` exceptions for inserts into `observability.otel_logs` around
`2026-10-18 20:58:53–20:58:54`.

This confirms telemetry inserts failed under memory pressure. It does
**not** prove that these errors caused every Collector restart or that
trace export itself failed.

### ClickHouse pod resources

  ---------------------------------------------------------------------------------
  Resource / setting                                                          Value
  ---------------------------------------- ----------------------------------------
  Kubernetes memory request                                                   `1Gi`

  Kubernetes memory limit                                                     `2Gi`

  CPU request                                                                `250m`

  CPU limit                                                                     `1`

  `max_server_memory_usage`                       `1932735283` bytes, approximately
                                                                         `1.80 GiB`

  `max_server_memory_usage_to_ram_ratio`                                      `0.9`

  `memory_worker_use_cgroup`                                                    `1`
  ---------------------------------------------------------------------------------

The explicit ClickHouse server memory limit is approximately 1.8 GiB,
close to the container's 2 GiB limit. The Kubernetes memory request is a
scheduling request, not a guarantee of that amount of physical RAM.

### Collector restart observations

The ClickHouse-facing Collector pod showed restart count `75`. The
previous container state showed termination reason `Error` and exit code
`2`; the current container was Running/Ready when inspected.

The restart count and memory errors are both observed facts, but the
causal relationship is unconfirmed. The captured previous logs included
startup information and a warning about a legacy telemetry resource
configuration format; the captured lines did not establish that warning
as the restart cause.

### Background merge observations

Two snapshots of `system.merges` were observed at different times:

-   `observability.otel_metrics_sum`: approximately `33.08 MiB` merge
    memory, progress `1` (100%).
-   `system.metric_log`: approximately `1.43 GiB` merge memory, progress
    approximately `0.1247` (12.47%).

A later query returned no active merges. These snapshots do not prove
that the `system.metric_log` merge caused the `otel_logs` insert
failures.

The `system.metric_log` table's active parts were measured as follows:

    Active parts   On-disk size                              Rows
  -------------- -------------- ---------------------------------
              12   `163.85 MiB`   approximately `702.21 thousand`

Disk size and in-memory merge usage are different measures. The merge's
reported memory use was much higher than the table's on-disk size at
that moment.

### Operational impact

The errors indicate that ClickHouse telemetry ingestion was not fully
healthy during the observed period. Retries can cause ingestion delays;
actual loss or queue exhaustion must be verified using Collector
exporter queue/retry metrics and by comparing received versus stored
telemetry.

No configuration changes, table truncations, or data deletions were
performed during this investigation.

## 9. Troubleshooting commands

### Check ClickHouse memory settings

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT name, value
FROM system.server_settings
WHERE name ILIKE '%memory%'
ORDER BY name
FORMAT PrettyCompact
"
```

### Check pod requests and limits

``` bash
kubectl get pod -n clickhouse clickhouse-clickhouse-0-0-0 \\
  -o jsonpath='{range .spec.containers[*]}Container: {.name}{"\\n"}Requests: {.resources.requests}{"\\n"}Limits: {.resources.limits}{"\\n\\n"}{end}'
```

### Inspect currently running queries

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT
    elapsed,
    formatReadableSize(memory_usage) AS memory,
    query_id,
    left(query, 180) AS query
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 10
FORMAT PrettyCompact
"
```

`system.processes` only shows queries running at query time. A
low-memory snapshot does not rule out earlier spikes.

### Inspect recent memory-limit exceptions

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT
    event_time,
    type,
    query_duration_ms,
    formatReadableSize(memory_usage) AS memory,
    exception_code,
    left(exception, 250) AS error,
    left(query, 120) AS query
FROM system.query_log
WHERE event_time >= now() - INTERVAL 24 HOUR
  AND exception_code = 241
ORDER BY event_time DESC
LIMIT 10
FORMAT Vertical
"
```

### Inspect active merges

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT
    database,
    table,
    elapsed,
    formatReadableSize(memory_usage) AS memory,
    progress
FROM system.merges
ORDER BY memory_usage DESC
FORMAT PrettyCompact
"
```

### Inspect ClickHouse trace-table size

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    sum(rows) AS rows
FROM system.parts
WHERE active
  AND database = 'observability'
  AND table = 'otel_traces'
GROUP BY table
FORMAT PrettyCompact
"
```

### Inspect Collector logs

``` bash
kubectl logs -n monitoring \\
  otel-collector-opentelemetry-collector-59bd9c8d5-4khv7 \\
  --since=24h 2>&1 \\
  | grep -Ei 'fatal|panic|error|failed|invalid|exporter|retry|shutdown' \\
  | tail -60
```

## 10. Findings summary

  -----------------------------------------------------------------------
  Area                    Finding                 Caveat
  ----------------------- ----------------------- -----------------------
  Ingestion               Both collectors         Confirmed for this test
                          accepted the controlled request
                          OTLP test request with  
                          HTTP 200                

  Retrieval               Tempo API returned the  Confirmed
                          test trace; ClickHouse  
                          query found the trace   
                          ID                      

  Span rows               ClickHouse had two      Cause unknown
                          identical rows for one  
                          unique span ID          

  Latency                 Median: ClickHouse      Operations are not
                          `count()` 6.129 ms;     equivalent
                          ClickHouse `SELECT *`   
                          14.835 ms; Tempo API    
                          9.022 ms                

  ClickHouse trace table  `31.93 KiB`, 5 total    Measured at that time
                          rows                    

  Tempo trace storage     Actual trace-only disk  Pending
                          usage not measured      

  ClickHouse memory       Code `241` insert       Confirmed
                          errors were observed    
                          for logs and metrics    

  Collector restarts      Restart count 75 and    Cause not established
                          prior exit code 2 were  
                          observed                

  Production readiness    Not established by this More testing required
                          small POC               
  -----------------------------------------------------------------------

## 11. Recommendations and remaining work

1.  **Investigate duplicate ingestion.** Compare collector pipelines,
    retry behavior, repeated requests, and any duplicate exporters. Do
    not delete duplicate rows until the cause and retention requirements
    are understood.
2.  **Stabilize ClickHouse ingestion.** Review memory use, active
    merges, telemetry volume, Collector batching/retry settings, and
    available Kubernetes node memory. Change memory limits or ingestion
    settings only after reviewing the complete configuration and
    available resources.
3.  **Confirm trace parity.** Use a larger controlled dataset with known
    trace and span counts; verify that both backends store the expected
    number of unique spans.
4.  **Run equivalent latency tests.** Compare trace-ID lookup with
    trace-ID lookup on the same trace set. Record at least 20--30
    repetitions, median, p95, response size, and error rate. Run warm-up
    requests separately.
5.  **Measure storage fairly.** Measure trace-only data in both systems
    or clearly label total backend storage separately. Do not compare
    ClickHouse's total PVC usage with Tempo's PVC capacity as if they
    were equivalent.
6.  **Measure resources.** Capture CPU and memory under the same
    ingestion and query workload, including background activity and
    Collector resources.
7.  **Document workload parameters.** Record trace count, spans per
    trace, attribute cardinality, retention, ingest rate, query pattern,
    resource limits, and storage configuration.

## 12. Conclusion

The POC confirms that ClickHouse and Tempo both accepted and returned
the controlled test trace. ClickHouse provides SQL access to telemetry
stored alongside logs and metrics, which can support cross-signal
analytics. Tempo is purpose-built for distributed tracing and exposes
trace lookup through its API.

The current evidence does not justify naming an overall performance or
storage winner. The dataset is small, the measured operations are not
equivalent, Tempo's trace-only storage has not been measured, and
ClickHouse showed memory-limit ingestion errors plus duplicate rows for
the test span.

**Preliminary recommendation:** Consider ClickHouse when consolidated
SQL analytics across logs, metrics, and traces is a key requirement.
Consider Tempo when the primary requirement is a dedicated
distributed-tracing backend. Make the final decision after equivalent
workload testing, duplicate-ingestion investigation, trace-only storage
measurement, and resource benchmarking.

## Appendix: quick verification

Check the test trace in ClickHouse:

``` bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \\
  clickhouse-client --query "
SELECT count() AS total_rows, uniqExact(SpanId) AS unique_spans
FROM observability.otel_traces
WHERE TraceId = 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa'
"
```

Check the test trace in Tempo (with port-forward active):

``` bash
curl -s http://localhost:3200/api/traces/aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```
