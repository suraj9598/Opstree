# ClickHouse vs Loki — Verified POC Comparison

## 1. Purpose

This README records the ClickHouse and Loki comparison work completed in the Kubernetes observability POC. It separates **measurements recorded in the POC notes** from items that still need verification. It does not treat screenshot-only panel values as verified benchmark results.

ClickHouse stores log records in a table and supports SQL queries. Loki stores log streams and supports LogQL queries. Grafana is used to view and compare the systems.

## 2. Architecture recorded for the POC

### ClickHouse logs

```text
Kubernetes logs -> Fluent Bit -> HTTP -> ClickHouse
                                       |
                                       v
                       observability.kubernetes_logs
                                       |
                                     Grafana
```

### Loki logs

```text
Kubernetes logs -> configured log collection -> Loki -> Grafana
```

The ClickHouse pipeline recorded in the POC uses Fluent Bit to send Kubernetes logs to ClickHouse. The exact Loki collection path and whether its collection configuration is equivalent to the ClickHouse path must be checked in the deployed configuration before claiming an identical workload.

## 3. Environment

| Item | Recorded configuration |
|---|---|
| Kubernetes | Local Kind cluster |
| ClickHouse namespace | `clickhouse` |
| Traditional observability namespace | `traditional-observability` |
| ClickHouse log table | `observability.kubernetes_logs` |
| Loki NodePort used during the POC | `32446` |
| Visualization | Grafana |

Resource names and port-forward sessions may change after restarts. Verify current pod/service status before using commands in a live demo.

## 4. Measurements recorded in the POC notes

The following values were recorded during previous checks. They are snapshots, not guaranteed current values.

| Measurement | Recorded result | Scope / limitation |
|---|---:|---|
| ClickHouse `observability.kubernetes_logs` table size | About **58.17 MiB** in a later check | Table-level data size from ClickHouse table parts |
| ClickHouse Kubernetes log rows | About **3,598,050 rows** in a later check | Row count for `observability.kubernetes_logs` |
| ClickHouse full PVC directory usage | Approximately **2.8–3.2 GiB** across recorded checks | Full ClickHouse storage directory/PVC scope, not only the log table |
| Loki `/data` directory usage | Approximately **66–73.9 MiB** across recorded checks | Directory-size snapshots; exact value changed between checks |
| Loki PVC usage | Approximately **72–73 MiB** in recorded checks | PVC usage snapshot |

### How to interpret these values

- The ClickHouse log-table size and the complete ClickHouse PVC usage measure different scopes.
- Loki `/data` directory size and Loki PVC usage are also separate measurements.
- Do not compare ClickHouse's full PVC usage directly with only the Loki `/data` directory.
- The values changed between checks, so take fresh measurements if a current figure is needed.

The ClickHouse table size and row count above are the latest specific table measurements recorded in the available POC notes. The storage directory values are approximate snapshots and should be refreshed before being used as current values.

## 5. Verified ClickHouse queries

Run these queries in the ClickHouse client. The pod name used in earlier POC commands was `clickhouse-clickhouse-0-0-0`; check the current name before running a pod-specific command.

### Open ClickHouse client

```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client
```

### Read recent logs

```sql
SELECT *
FROM observability.kubernetes_logs
ORDER BY timestamp DESC
LIMIT 10;
```

### Count log rows

```sql
SELECT count() AS total_logs
FROM observability.kubernetes_logs;
```

### Check table size and row count

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    sum(rows) AS rows
FROM system.parts
WHERE active = 1
  AND database = 'observability'
  AND table = 'kubernetes_logs'
GROUP BY database, table;
```

This measures active table parts for the `kubernetes_logs` table. It does not represent the full ClickHouse PVC.

### Check latest stored log timestamp

```sql
SELECT
    max(timestamp) AS latest_log,
    count() AS total_logs
FROM observability.kubernetes_logs;
```

This checks the newest timestamp stored in the table and the current row count. Run it again before a presentation if you need fresh data.

## 6. Loki query recorded for verification

In Grafana Explore, select the Loki datasource and query labels that exist in the current data. The POC notes recorded these label names:

- `container`
- `job`
- `namespace`
- `pod`

Example selector, **only if this namespace label value exists in Loki data**:

```logql
{namespace="traditional-observability"}
```

The selector is an example, not proof that the namespace currently has matching logs. If it returns no data, inspect the available label values in Grafana Explore.

## 7. Grafana comparison dashboard — current interpretation

The screenshots show that panels were created for storage, ingestion rate, latency, and query rate. However, the screenshots alone do not expose all underlying query definitions. Therefore, panel values are **not listed here as verified benchmark results**.

Before using a panel value in a report, open the panel editor and verify:

- the exact query;
- what the query measures;
- the unit and legend;
- the time range;
- whether the value is a last observation, average, or rate;
- whether the ClickHouse and Loki queries measure equivalent work.

### Storage panels

The panel values visible in screenshots can only be reported accurately after confirming whether each query measures a table, directory, PVC, or another scope. Use the table-level ClickHouse query in Section 5 and a matching, clearly documented Loki storage measurement.

### Ingestion-rate panels

A value labelled `ops/s` is not self-explanatory. Confirm whether it counts log records, output operations, or another counter. Only compare rates if both queries count equivalent events over compatible time windows.

### Latency panels

ClickHouse Insert latency, ClickHouse SELECT latency, and Loki range-query latency refer to different operations. They cannot be treated as a direct apples-to-apples comparison without equivalent test cases and confirmed query definitions.

### Query-rate panels

SELECT queries per second and Loki range queries per second represent query activity. They do not represent query latency or log ingestion throughput.

## 8. Kubernetes health checks

```bash
kubectl get pods -n clickhouse
kubectl get pods -n traditional-observability
kubectl get pvc -n clickhouse
kubectl get pvc -n traditional-observability
```

These commands show pod and PVC status. They do not by themselves measure table size, query latency, or equivalent log volume.

## 9. What the POC supports concluding

### Supported by the recorded POC notes

- Kubernetes log records were present in `observability.kubernetes_logs`.
- A later check recorded approximately 3,598,050 rows and about 58.17 MiB for the ClickHouse Kubernetes log table.
- ClickHouse full storage usage was measured separately from the log-table size.
- Loki was running during the recorded checks, and its data directory/PVC usage was inspected.
- Loki queries returned log entries for selected queries/time ranges.
- Grafana comparison panels were created.

### Not established by the current evidence

- That one system is definitively faster.
- That one system is definitively more storage-efficient under equal conditions.
- That both systems received exactly the same log records.
- That screenshot ingestion rates or latency values are comparable.
- That the latest screenshot values remain current.

## 10. Recommended next validation

- [ ] Run fresh ClickHouse table-size and row-count queries.
- [ ] Measure Loki storage with a clearly defined scope.
- [ ] Confirm both systems are receiving equivalent log records.
- [ ] Open every Grafana panel and record the exact query and unit.
- [ ] Re-run comparable query tests with the same time range and filters.
- [ ] Record the timestamp of each measurement.
- [ ] Save the dashboard and update this README with verified results only.

## 11. Presentation-ready conclusion

The POC demonstrates that Kubernetes logs can be queried in ClickHouse using SQL and in Loki using LogQL, and that Grafana panels have been created to compare their observed behavior. The recorded ClickHouse log-table measurement was approximately 58.17 MiB for about 3.60 million rows. ClickHouse's complete PVC usage was measured separately and is not equivalent to the log-table size. Loki storage values also varied between checks, so fresh measurements should be taken before presenting a current storage comparison.

The available screenshots show storage, ingestion-rate, latency, and query-rate panels, but their query definitions must be verified before treating the values as comparable benchmark results. The accurate conclusion is that the POC demonstrates basic log storage/querying and provides an initial dashboard; a definitive performance comparison requires equivalent workloads, verified panel queries, and fresh measurements.

---
**Document status:** Verified-scope comparison README. Screenshot-only values are intentionally excluded as verified benchmark results until their panel queries and measurement scopes are confirmed.
