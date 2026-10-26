# Cerberus-Based Loki Query Routing to ClickHouse for Grafana

**Last updated:** 2026-10-26 | **Cluster context:** `kind-kind-clickhouse-poc` | **Cerberus:** chart `0.18.0`, app `1.22.0` | **ClickHouse:** `26.9.4.3`

## Table of Contents

1. [Objective and Summary](#1-objective-and-summary)
2. [POC Goals and Scope](#2-poc-goals-and-scope)
3. [Environment](#3-environment)
4. [Architecture](#4-architecture)
5. [Existing Observability Pipelines](#5-existing-observability-pipelines)
6. [ClickHouse Data Layer](#6-clickhouse-data-layer)
7. [What Cerberus Is](#7-what-cerberus-is)
8. [Cerberus Deployment](#8-cerberus-deployment)
9. [NGINX Query/Write Router](#9-nginx-querywrite-router)
10. [Grafana Access and Reverse SSH Tunnel](#10-grafana-access-and-reverse-ssh-tunnel)
11. [Validation Results](#11-validation-results)
12. [LogQL to SQL Translation Observed](#12-logql-to-sql-translation-observed)
13. [Known Issues and Gaps](#13-known-issues-and-gaps)
14. [Risks and Limitations](#14-risks-and-limitations)
15. [Troubleshooting Notes](#15-troubleshooting-notes)
16. [Next Steps](#16-next-steps)
17. [Current Status](#17-current-status)
18. [Conclusion and Recommendation](#18-conclusion-and-recommendation)
19. [Appendix: Verification Commands](#19-appendix-verification-commands)

---

## 1. Objective and Summary

**Objective:** keep the existing Grafana Loki data source (and its dashboards) unchanged, and make Loki-style queries execute as SQL on ClickHouse. Cerberus is used as a Loki-compatible, read-only query layer between Grafana and ClickHouse. The existing Loki log-ingestion path is preserved.

**Verdict:** *partially feasible, with conditions* (details in [Section 18](#18-conclusion-and-recommendation)).

**Result:** the existing Grafana datasource `loki-1` (`http://127.0.0.1:32446`) now returns logs through `NGINX → Cerberus → ClickHouse (observability.otel_logs)`. ClickHouse `system.query_log` shows the translated SQL (namespace predicate, time window, `ORDER BY Timestamp DESC`, `LIMIT`). Push requests are routed to Loki. Compatibility is proven only for the tested query shapes. A legacy-label gap (`namespace` vs `k8s_namespace_name`) and a routing issue for label-values requests are open (see [Section 13](#13-known-issues-and-gaps)).

### Request flow

```text
Query:    Grafana → Reverse SSH Tunnel → NGINX Router → Cerberus → ClickHouse
Response: ClickHouse → Cerberus → NGINX Router → Reverse SSH Tunnel → Grafana
Write:    Fluent Bit → loki-stack (internal Service) → Loki      (unchanged)
```

1. **Grafana** sends Loki API requests to `http://127.0.0.1:32446` (datasource URL, unchanged).
2. **Reverse SSH tunnel** forwards that port to the Kubernetes NGINX NodePort `172.19.0.2:30080`.
3. **NGINX** sends supported Loki read/query paths to Cerberus. `/loki/api/v1/push` and all other paths go to Loki.
4. **Cerberus** parses LogQL and generates ClickHouse SQL.
5. **ClickHouse** executes the SQL on `observability.otel_logs` and returns rows.
6. **Grafana** renders the logs in Explore or dashboards.

## 2. POC Goals and Scope

**Goals**

- Document the current logs, metrics and traces pipelines.
- Evaluate Cerberus as a Loki-compatible query layer for ClickHouse.
- Keep the Grafana Loki data source and dashboards unchanged where technically feasible.
- Preserve the current log-ingestion path while testing query routing.
- Validate compatibility before changing any Service or data-source endpoint.

**In scope:** logs and Loki-compatible read queries.

**Out of scope:** metrics (VictoriaMetrics) and traces (Tempo) remain unchanged. Cerberus also supports PromQL and TraceQL upstream, but they were not evaluated. Ingestion through Cerberus is not part of the design: Cerberus is read-only.

## 3. Environment

| Namespace | Components | Role |
|---|---|---|
| `traditional-observability` | Loki, Tempo, VictoriaMetrics, vmagent, Fluent Bit, OTel Collector, Loki storage exporter | Traditional stack |
| `logging` | Fluent Bit DaemonSet | Container logs → ClickHouse |
| `monitoring` | OTel Collector, OTel logs agent, node-exporter, kube-state-metrics | Infra telemetry and OTel log pipeline |
| `clickhouse` | ClickHouse, Keeper, services, PVCs | Telemetry storage |
| `gigapipe` | Gigapipe pod and service | Additional Loki-compatible log destination |
| `hyperdx` | HyperDX, MongoDB, HyperDX OTel Collector | Separate HyperDX POC |
| `cerberus` | Cerberus | Loki-compatible query gateway (this POC) |
| `log-router` | NGINX | Path-based query/write router (this POC) |
| `clickhouse-operator-system` | ClickHouse Operator | ClickHouse cluster management |

Grafana runs on a **separate VM** and reaches the cluster only through reverse SSH tunnels from the laptop that hosts the Kind cluster.

**Services relevant to this POC**

| Service | Namespace | Endpoint |
|---|---|---|
| `loki-stack` | `traditional-observability` | ClusterIP `3100` (Fluent Bit write path) |
| `loki-external` | `traditional-observability` | NodePort `32446` → `3100` |
| `tempo-external` | `traditional-observability` | NodePort `31000` → `3200` |
| `vm-external` | `traditional-observability` | NodePort `31720` → `8428` |
| `clickhouse-clickhouse-headless` | `clickhouse` | `8123` (HTTP), `9000` (native), `9363` |
| `clickhouse-http` | `clickhouse` | NodePort `30081` → `8123` |
| `cerberus` | `cerberus` | ClusterIP `8080` |
| `log-router-nginx` | `log-router` | NodePort `30080` → `8080` |

## 4. Architecture

<img width="4500" height="1452" alt="mermaid-diagram" src="https://github.com/user-attachments/assets/0690636a-b999-4a96-8cb7-239c8c055f0d" />

Solid arrows are paths confirmed from inspected configuration. Dashed arrows are not proof of a working deployment. VictoriaMetrics ingestion has not been verified end to end.

<img width="976" height="372" alt="Screenshot from 2026-10-26 14-51-10" src="https://github.com/user-attachments/assets/c750a7bb-2904-41d8-87a4-6e6c72f44e4f" />


**Design decision:** The `loki-external` selector and Fluent Bit's internal `loki-stack` destination were deliberately left untouched. Repointing the Loki Service at Cerberus would break log ingestion, because Cerberus does not accept pushes. A path-aware proxy (NGINX) in front lets reads go to Cerberus while writes continue to Loki.

## 5. Existing Observability Pipelines

### 5.1 Fluent Bit → Loki and Gigapipe

ConfigMap `fluent-bit-traditional` (`traditional-observability`). It tails `/var/log/containers/*.log`, excludes Fluent Bit's own logs, applies Docker/CRI multiline parsing, adds Kubernetes metadata, and has two `loki` outputs:

1. Loki: `loki-stack.traditional-observability.svc.cluster.local:3100`
2. Gigapipe: `gigapipe.gigapipe.svc.cluster.local:3100`

Labels sent: job, namespace, pod, container. This is why Loki's native label is `namespace`, while Cerberus exposes OTel-style labels (see Issue 1 in Section 13).

### 5.2 Traditional OpenTelemetry Collector

ConfigMap `otel-collector-traditional-opentelemetry-collector`. Receivers: OTLP gRPC `4317`, OTLP HTTP `4318`, Jaeger, Zipkin, and a Prometheus receiver scraping its own metrics on `8888`.

| Pipeline | Flow |
|---|---|
| Logs | OTLP → memory limiter → batch → `debug` exporter |
| Metrics | OTLP + Prometheus → memory limiter → batch → `debug` exporter |
| Traces | OTLP → batch → OTLP/gRPC to `tempo.traditional-observability.svc.cluster.local:4317` |

Its logs and metrics pipelines do not export to Loki or VictoriaMetrics. Other independent telemetry paths may exist.

### 5.3 Fluent Bit → ClickHouse `kubernetes_logs`

ConfigMap `fluent-bit` (`logging`). Output: host `clickhouse-clickhouse-headless.clickhouse.svc.cluster.local`, port `8123` (HTTP), table `observability.kubernetes_logs`, format `JSONEachRow`. Columns: `timestamp DateTime`, `namespace`, `pod`, `container`, `node`, `message`.

### 5.4 OTel file logs → ClickHouse `otel_logs` (used by Cerberus)

ConfigMap `otel-logs-opentelemetry-collector-agent` (`monitoring`).

- Receiver: `file_log` on `/var/log/pods/*/*/*.log`, collector's own path excluded, container parser, `start_at: end`.
- Exporter: ClickHouse, endpoint `tcp://clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000`, database `observability`, table `otel_logs`, `async_insert: true`, `create_schema: true`, user `default`.
- Pipeline: `file_log` → memory limiter → batch → ClickHouse.
- Metrics and traces pipelines in this ConfigMap use `debug`, not ClickHouse.

### 5.5 OTel metrics and traces → ClickHouse

ConfigMap `otel-collector-opentelemetry-collector` (`monitoring`). Metrics: Prometheus receiver (node-exporter, kube-state-metrics, cAdvisor) → ClickHouse exporter. Traces: OTLP (plus Jaeger/Zipkin receivers) → ClickHouse exporter. Logs: OTLP → `debug`. Live delivery per table still needs verification.

### 5.6 Loki endpoint verification

`loki-external` was confirmed to route to pod `loki-stack-0` (`10.244.2.8:3100`, node `kind-clickhouse-poc-worker`, `1/1 Running`, 31 restarts at the time of check). The restart count is worth investigating separately; it does not by itself indicate an outage. Loki's `/ready` returns `200`, while `/readyz` returns `404` (endpoint-specific, not a health problem).

## 6. ClickHouse Data Layer

**Pods and storage:** ClickHouse pod `clickhouse-clickhouse-0-0-0` (PVC `5Gi`), Keeper pod `clickhouse-keeper-keeper-0-0` (PVC `2Gi`), managed by the ClickHouse Operator.

**Tables in `observability`:** `kubernetes_logs`, `kubernetes_logs_real_test`, `kubernetes_logs_test`, `logs`, `metrics`, `otel_logs`, `otel_metrics_exponential_histogram`, `otel_metrics_gauge`, `otel_metrics_histogram`, `otel_metrics_sum`, `otel_metrics_summary`, `otel_traces`, `otel_traces_trace_id_ts`, `otel_traces_trace_id_ts_mv`.

### 6.1 Why `otel_logs` and not `kubernetes_logs`

Cerberus reads the OpenTelemetry ClickHouse exporter schema. `CERBERUS_SCHEMA_LOGS_TABLE` changes the table name, but there is no column-name override.

| | `kubernetes_logs` | `otel_logs` |
|---|---|---|
| Time column | `timestamp DateTime` | `Timestamp DateTime64(9)` |
| Log text | `message` | `Body` |
| Metadata | `namespace`, `pod`, `container`, `node` columns | `ResourceAttributes`, `ScopeAttributes`, `LogAttributes`, materialized `k8s.*` columns |
| Other | none | `TraceId`, `SpanId`, `TraceFlags`, `SeverityText`, `SeverityNumber`, `ServiceName` |
| Cerberus-compatible | No (expected columns missing) | Yes |

No compatibility view and no ClickHouse DDL was required.

### 6.2 `otel_logs` data snapshot (2026-10-23, read-only checks)

| Check | Result |
|---|---|
| Latest `Timestamp` | `2026-10-23 11:47:21.278312092` |
| Total rows | `2,744,925` (later check: `2,746,568`) |
| Rows in previous 15 minutes | `10,236` |
| Rows with non-empty `Body` | `2,715,447` (~98.9%) |
| `ServiceName` | empty for all rows checked |

Sample rows showed populated Kubernetes resource attributes. `LogAttributes` keys: `log.file.path`, `log.iostream`, `logtag`. Row counts are point-in-time observations, not guaranteed current values.

### 6.3 ClickHouse authentication

The `default` user uses `no_password` with `host_ip: ['::/0']`. Connectivity from inside the cluster was confirmed with a temporary pod (`SELECT version(), 1` returned `26.9.4.3  1`). This is acceptable for a POC only (see Section 14).

## 7. What Cerberus Is

Cerberus ([tsouza/cerberus](https://github.com/tsouza/cerberus)) is a **stateless, read-only query gateway** for ClickHouse. It exposes endpoints compatible with Grafana's Prometheus, Loki and Tempo data sources and translates supported queries into ClickHouse SQL. This POC uses only the Loki (logs) head.

| Cerberus does | Cerberus does not |
|---|---|
| Accept supported Loki query API requests | Store logs or act as a database |
| Translate supported LogQL to ClickHouse SQL | Ingest from Fluent Bit or replace the OTel Collector |
| Return Loki-shaped responses to Grafana | Guarantee full LogQL parity for every dashboard |
| Sit behind a reverse proxy | Redirect Grafana by itself (routing must reach it) |

### Query lifecycle

1. Cerberus receives a Loki request, e.g. `/loki/api/v1/query_range`.
2. It parses the LogQL expression and the time range/step parameters.
3. It maps the expression to the configured schema and generates SQL.
4. ClickHouse filters and scans the data and returns rows or aggregates.
5. Cerberus converts the result to a Loki-compatible response.
6. Grafana renders it. HTTP `200` alone does not prove the result is semantically identical to Loki: labels and log lines must also be compared.

### Benefits and trade-offs

**Potential benefits:** reuse existing Loki queries and Grafana panels; keep reads separate from ingestion; use ClickHouse as the storage/query engine; test incrementally behind a proxy.

**Trade-offs:** compatibility is endpoint-by-endpoint and query-by-query; schema and field population affect correctness; one more component and one more hop to operate and secure.

## 8. Cerberus Deployment

### 8.1 Install

```bash
kubectl create namespace cerberus

helm install cerberus oci://ghcr.io/tsouza/cerberus/charts/cerberus \
  --version 0.18.0 --namespace cerberus \
  --set replicaCount=1 \
  --set 'clickhouse.addr[0]=clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000' \
  --set clickhouse.database=observability \
  --set clickhouse.username=default \
  --set clickhouse.protocol=native \
  --set autoCreate.schema=false \
  --set autoCreate.database=false \
  --set-string config.CERBERUS_SCHEMA_LOGS_TABLE=otel_logs
```

Chart defaults worth knowing: deployment mode `monolith`, default replicas `2` (overridden to `1`), Service `ClusterIP` on port `8080`, extra `CERBERUS_*` variables via the chart's `config` map. A separate namespace keeps Cerberus isolated from ClickHouse and Keeper without affecting connectivity. The chart was first rendered with `helm template` and the key values were confirmed before installing.

### 8.2 Effective configuration (`cerberus-env` ConfigMap)

| Setting | Value | Purpose |
|---|---|---|
| `CERBERUS_CH_ADDR` | `clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000` | ClickHouse native endpoint |
| `CERBERUS_CH_DATABASE` | `observability` | Database with telemetry tables |
| `CERBERUS_CH_PROTOCOL` | `native` | Native protocol |
| `CERBERUS_CH_USERNAME` | `default` | ClickHouse user (POC only) |
| `CERBERUS_SCHEMA_LOGS_TABLE` | `otel_logs` | Logs table to query |
| `CERBERUS_ADMIT_LOKI` | `true` | Loki request admission control |
| `CERBERUS_HTTP_ADDR` | `:8080` | HTTP listener |
| `CERBERUS_LOG_LEVEL` / `CERBERUS_LOG_FORMAT` | `info` / `json` | Application logging |

Image: `ghcr.io/tsouza/cerberus:1.22.0`. Self-telemetry export is disabled (OTLP endpoint empty). Credentials, if configured later, should come from Kubernetes Secrets, not from values files or this README.

### 8.3 Verification

| Check | Result |
|---|---|
| `kubectl get pods,svc -n cerberus` | Pod `1/1 Running`, 0 restarts; Service `cerberus` ClusterIP on `8080` |
| `GET /readyz` | `200`: `{"clickhouse":"ok","schema":"ready","heads":{"loki":"closed","prom":"closed","tempo":"closed"}}` |
| Startup logs | Version `1.22.0`, ClickHouse `26.9.4.3`, requirements check passed (`native_rate: true`), admission control enabled, query heads `prom,loki,tempo`, listener ready; no errors |

Helm `STATUS: deployed` only confirms the release was installed. The `closed` values under `heads` did not prevent Loki API calls from succeeding; their exact meaning (likely admission-control state) was not confirmed from upstream documentation.

## 9. NGINX Query/Write Router

NGINX (`nginx:stable`, one replica) runs in namespace `log-router`, exposed on NodePort `30080`. It was deployed in the cluster rather than on the laptop; host port `8080` on the laptop is used by an unrelated Docker container, so NGINX listens on container port `8080` inside its own pod network namespace.

### 9.1 Routing rules

```nginx
upstream loki_backend     { server loki-external.traditional-observability.svc.cluster.local:3100; }
upstream cerberus_backend { server cerberus.cerberus.svc.cluster.local:8080; }

server {
  listen 8080;

  location = /ready { access_log off; default_type text/plain; return 200 "ready\n"; }

  # Write path stays on Loki
  location = /loki/api/v1/push { proxy_pass http://loki_backend; }

  # Read/query paths go to Cerberus
  location ~ ^/loki/api/v1/(query|query_range|labels|label/|series|index/stats|index/volume|index/volume_range|detected_fields|detected_labels)(/|$) {
    proxy_pass http://cerberus_backend;
  }

  # Everything else falls back to Loki
  location / { proxy_pass http://loki_backend; }
}
```

| Rule | Behavior |
|---|---|
| `= /ready` | Answered by NGINX itself (`200 ready`) |
| `= /loki/api/v1/push` | Always Loki (write path) |
| Regex (query, query_range, labels, label, series, index/*, detected_*) | Cerberus |
| `/` (anything else, e.g. `tail`, `/ready` of Loki, unlisted paths) | Loki |

> **Issue found during review (not yet fixed or tested):** the alternative `label/` must be followed by `/` or end-of-string, so `/loki/api/v1/label/<name>/values` does **not** match and falls back to Loki. Proposed fix: replace `label/` with `label(/|$)`. See Issue 2 in Section 13.

### 9.2 Deployment manifest summary

- ConfigMap `log-router-nginx` mounted at `/etc/nginx/conf.d/default.conf` (`subPath: default.conf`, read-only).
- Deployment `log-router-nginx`: 1 replica, `nginx:stable`, `imagePullPolicy: IfNotPresent`, container port `8080`.
- Readiness probe: HTTP GET `/ready` on `8080` (initial delay 3s, period 5s). Liveness: TCP `8080` (initial delay 10s, period 10s).
- Service `log-router-nginx`: NodePort `30080` → `8080`.
- Manifests: `log-router-nginx-configmap.yaml`, `log-router-nginx-deployment.yaml`.

```bash
kubectl apply -f log-router-nginx-configmap.yaml
kubectl apply -f log-router-nginx-deployment.yaml
kubectl rollout status deployment/log-router-nginx -n log-router
kubectl get pods,svc -n log-router -o wide
```

The first pod was `Running` but `0/1 Ready` because the initial probe requested `/` (HTTP `404`). Adding the `/ready` location and pointing the probe at it fixed this; the pod then became `1/1 Running` with 0 restarts.

## 10. Grafana Access and Reverse SSH Tunnel

### 10.1 Current tunnel layout

Reverse SSH tunnels run from the laptop to the Grafana VM. The existing Loki port now points to NGINX; the VictoriaMetrics and Tempo forwards were retained in the replacement session.

```bash
ssh -o ExitOnForwardFailure=yes -i <key.pem> -N \
  -R 127.0.0.1:32446:172.19.0.2:30080 \
  -R 127.0.0.1:31720:172.19.0.2:31720 \
  -R 127.0.0.1:31000:172.19.0.2:31000 \
  ubuntu@<grafana-vm-ip>
```

| Grafana VM loopback port | Destination | Purpose |
|---|---|---|
| `32446` | `172.19.0.2:30080` (NGINX) | Grafana Loki datasource URL (unchanged) |
| `31720` | `172.19.0.2:31720` | VictoriaMetrics |
| `31000` | `172.19.0.2:31000` | Tempo |

**Separate, independent tunnels (must be preserved):**

| Grafana VM port | Destination | Purpose |
|---|---|---|
| `8123` | `172.19.0.2:30081` | ClickHouse HTTP (ClickHouse datasource) |
| `18081` | laptop `127.0.0.1:8081` → `kubectl port-forward svc/cerberus 8081:8080` | Direct Cerberus testing |

On the Grafana VM the listeners show up as `sshd` (the SSH server owns reverse-forward ports); the SSH client process and its `-R` arguments are on the laptop.

### 10.2 How the switch was done

1. A temporary tunnel on Grafana VM port `18082` → `172.19.0.2:30080` was created first. `/ready` and a `query_range` request through it succeeded, without touching any existing tunnel.
2. The old multi-forward SSH process was stopped. A stale `sshd` listener still held port `32446`; it was identified and cleared.
3. The replacement session (Section 10.1) was started with `ExitOnForwardFailure=yes` and ran without errors.
4. `/ready` and `query_range` on `127.0.0.1:32446` from the Grafana VM were verified.

### 10.3 Rollback

Restore the original mapping and restart the tunnel:

```bash
-R 127.0.0.1:32446:172.19.0.2:32446
```

No Loki Service selector, Fluent Bit output, or Grafana datasource setting was changed, so this is the only change to undo. Host IPs and key paths are intentionally omitted from this document.

## 11. Validation Results

### 11.1 Direct Cerberus API (via port-forward)

| Test | Result |
|---|---|
| `GET /loki/api/v1/labels` | `200`: `k8s_container_name`, `k8s_container_restart_count`, `k8s_namespace_name`, `k8s_pod_name`, `k8s_pod_uid` |
| `query_range` `{k8s_namespace_name="traditional-observability"}` (15 min, limit 5) | `success`, `streams`; logs from storage exporter, Loki and the traditional OTel Collector; labels include namespace/pod/container plus `log_file_path`, `log_iostream`, `logtag` |
| `query_range` with `\|= "GET /metrics"` | `success`; one stream (pod `loki-storage-exporter-...`, container `exporter`), five lines, HTTP `200` |

### 11.2 Loki vs Cerberus (same query shape)

```bash
# Loki
curl -sS -G 'http://127.0.0.1:32446/loki/api/v1/query_range' \
  --data-urlencode 'query={namespace="traditional-observability"} |= "GET /metrics"' \
  --data-urlencode "start=$(date -d '15 minutes ago' +%s)" \
  --data-urlencode "end=$(date +%s)" --data-urlencode 'limit=5'

# Cerberus
curl -sS -G 'http://127.0.0.1:8081/loki/api/v1/query_range' \
  --data-urlencode 'query={k8s_namespace_name="traditional-observability"} |= "GET /metrics"' \
  --data-urlencode "start=$(date -d '15 minutes ago' +%s)" \
  --data-urlencode "end=$(date +%s)" --data-urlencode 'limit=5'
```

Both returned `success` with five entries from the same pod containing `GET /metrics HTTP/1.1` and status `200`. The two queries ran at different times, so entries and timestamps were not identical. This confirms functional equivalence for this query shape only. Loki's `execTime` (~9.84 ms) is a backend statistic and must not be compared with an end-to-end HTTP duration measured for Cerberus; a fair benchmark needs a common window, repeated runs and consistent measurement.

### 11.3 NGINX routing

| Test | Result |
|---|---|
| `/ready` via Service and NodePort | `200 ready` |
| `labels` via NGINX | `200`, Cerberus-style labels |
| `query_range` via NGINX Service and NodePort | `success`, real streams |
| `POST /loki/api/v1/push` with `{}` | `204` (route is configured to Loki; this is not a real ingestion test) |
| Grafana VM → temporary tunnel `18082` → NGINX | `/ready` and `query_range` succeeded |
| Grafana VM `127.0.0.1:32446` after the switch | `/ready` and `query_range` succeeded |

The NGINX access log status alone does not identify which upstream produced the push response; the push-routing conclusion rests on the active configuration plus the direct Loki endpoint check.

### 11.4 Grafana Explore (existing datasource `loki-1`)

- `{k8s_namespace_name="clickhouse"}`: ClickHouse and Keeper logs displayed; logs-volume panel also returned data.
- `{k8s_namespace_name="traditional-observability"}`: OTel Collector and related logs displayed.

### 11.5 End-to-end evidence

**NGINX:** access logs show Grafana (`Grafana/13.2.1`, `loki/13.2.2`) requests returning `200`; one `query_range` response was `418324` bytes. NGINX also logged a buffering warning with upstream `http://10.96.165.8:8080/loki/api/v1/query_range?...`. That address matches Service `cerberus` (ClusterIP `10.96.165.8:8080`, endpoint `10.244.1.14:8080` at check time). The warning means a large upstream response was buffered to a temp file; the request itself returned `200`.

**ClickHouse:** after running the Explore query, `system.query_log` was inspected:

```sql
SYSTEM FLUSH LOGS;
SELECT event_time, query_duration_ms, read_rows,
       formatReadableSize(read_bytes) AS read_size,
       substring(query, 1, 500) AS query_preview
FROM system.query_log
WHERE event_time >= now() - INTERVAL 5 MINUTE
  AND type = 'QueryFinish'
  AND query ILIKE '%otel_logs%'
  AND query ILIKE '%Body%AS%Line%'
ORDER BY event_time DESC
LIMIT 10
FORMAT Vertical;
```

| ClickHouse event time | Duration | Rows read | Read size |
|---|---:|---:|---:|
| `2026-10-25 19:23:15` | 111 ms | 251,665 | 17.47 MiB |
| `2026-10-25 19:23:19` | 102 ms | 30,100 | 5.33 MiB |
| `2026-10-25 19:23:27` | 255 ms | 30,386 | 5.41 MiB |
| `2026-10-25 19:27:32` | 60 ms | 27,867 | n/a |

The `19:23:15` record occurred in the same second as the NGINX `query_range` request routed to `10.96.165.8:8080`. `read_rows` and `read_bytes` measure data scanned by ClickHouse, not log entries returned to Grafana. Some rows showed blank filter fields in an extraction query only because the SQL text did not contain the literal `PREWHERE`; read the full query text before classifying a record.

**Namespace-change test:** after switching the selector to `traditional-observability`, `system.query_log` showed the same statements with `PREWHERE (__otel_materialized_k8s.namespace.name = 'traditional-observability')`, confirming the predicate follows the Grafana selector.

**Correlation limitation:** no request ID ties an individual Grafana request to an individual SQL statement. Timestamp, endpoint, namespace predicate, time window and query shape are strong supporting evidence, not strict one-to-one proof. `kubectl logs -n cerberus deploy/cerberus --since=5m` returned no lines at `info` level; that does not show requests were not received, and whether Cerberus has a request/SQL debug option was not established.

## 12. LogQL to SQL Translation Observed

| Grafana / LogQL input | Observed ClickHouse behavior |
|---|---|
| `{k8s_namespace_name="clickhouse"}` | `PREWHERE (`__otel_materialized_k8s.namespace.name` = 'clickhouse')` on `otel_logs` |
| Explore time range | `WHERE Timestamp >= toDateTime64('...', 9) AND Timestamp < toDateTime64('...', 9)` |
| Log lines request | `SELECT Body AS Line, ... ORDER BY Timestamp DESC LIMIT 1000` |
| Logs volume panel | Aggregation using `count()` over time buckets |
| Label discovery | Query over distinct resource-attribute combinations, count and log-body size |
| `\|= "GET /metrics"` line filter | Returned correct lines through the API; the generated SQL was not captured in this POC |

`PREWHERE` on the materialized namespace column lets ClickHouse prune data before reading the other columns, which is why namespace-filtered queries stay cheap even on a table with millions of rows.

## 13. Known Issues and Gaps

| # | Issue | Detail | Proposed action |
|---|---|---|---|
| 1 | **Legacy `namespace` label** | Dashboard panel `ClickHouse vs Loki — Log Count Comparison` uses `sum(count_over_time({namespace="clickhouse"}[5m]))`. Through Cerberus this can return no data, while `{k8s_namespace_name="clickhouse"}` works. A direct Cerberus call to `/label/namespace/values` returned an empty list. Namespace is stored under the OTel key `k8s.namespace.name` (materialized column `__otel_materialized_k8s.namespace.name`); `ResourceAttributes['namespace']` is empty. This is a label/schema mismatch, not an ingestion failure. | **Option A:** change panel queries to `k8s_namespace_name` (test in a copy first; candidate: `sum(count_over_time({k8s_namespace_name="clickhouse"}[5m]))`, result not yet verified). **Option B:** add a `namespace` alias in Cerberus covering **both** matcher translation and label discovery/values. An earlier local matcher-only experiment passed its unit test but was never deployed, and a matcher alias alone would leave label menus and `/label/namespace/values` broken. |
| 2 | **NGINX label-values routing** | `label/<name>/values` does not match the router regex and falls back to Loki (Section 9.1). Grafana label browsers and variables may get Loki's answer instead of Cerberus's. | Change `label/` to `label(/|$)`, apply, and re-test `/loki/api/v1/label/k8s_namespace_name/values` through port `32446`. |
| 3 | **Large-response buffering** | NGINX buffered a ~418 KB upstream response to a temp file (request still `200`). | Tune `proxy_buffers` / `proxy_buffer_size` if it becomes noisy. |
| 4 | **ClickHouse datasource timeout** | The dashboard's ClickHouse query timed out: `net/http: timeout awaiting response headers` on the `8123` tunnel. Separate from the Loki/Cerberus path. | Check query cost and tunnel latency; raise the datasource timeout. |
| 5 | **Different source tables** | The dashboard's ClickHouse query reads `kubernetes_logs`; Cerberus reads `otel_logs`. Counts are not comparable until table, namespace field, time window and ingestion coverage are aligned. | Align both sides before comparing counts. |
| 6 | **Empty `ServiceName`** | All checked `otel_logs` rows have an empty `ServiceName`; service-based filtering or grouping is not useful. | Populate it in the collector if needed (for example from container or service attributes). |
| 7 | **Push path not truly tested** | Only an empty `{}` POST returned `204`. | Send a real Fluent Bit payload and confirm it appears in Loki. |
| 8 | **Request correlation** | No shared request ID across NGINX, Cerberus and ClickHouse. | Add a proxy-level request ID, or enable Cerberus request/debug logging if supported. |

## 14. Risks and Limitations

- **Compatibility is query-by-query.** Tailing, other detected-fields endpoints, metric queries beyond `count_over_time`, and dashboard transformations are untested.
- **Parity and latency** need identical queries, time windows and repeated runs before any claim.
- **Security:** the ClickHouse `default` user is passwordless and open to `::/0` in this POC. Use a dedicated read-only ClickHouse user for Cerberus, supply credentials via Kubernetes Secrets, and restrict network access.
- **Fragile access path:** the Grafana path depends on a laptop-hosted reverse SSH tunnel; NGINX and Cerberus run as single replicas. This is POC-grade and not highly available.
- **Extra hop:** NGINX and Cerberus add components to monitor, secure and debug.
- **Storage comparison:** ClickHouse PVC (`5Gi`) also holds metrics, traces and system data, so it is not comparable with the Loki PVC (`5Gi`) as log-only storage. A fair comparison needs equivalent datasets, windows and retention.
- **Not established:** production readiness, high availability, full dashboard compatibility, complete Loki replacement.

## 15. Troubleshooting Notes

| Symptom | Cause / resolution |
|---|---|
| Local port `8080` already in use when port-forwarding | Another process (a Docker container) owns it; port-forward Cerberus to `8081` instead. |
| `ssh -R` fails with a forwarding error on `32446` | A stale `sshd` listener remains on the Grafana VM; find it with `sudo ss -lntp \| grep 32446` and clear it before restarting the tunnel. Keep `ExitOnForwardFailure=yes` so failures are visible. |
| No SSH client process on the Grafana VM | Expected: the VM only shows `sshd` listeners; the SSH client runs on the laptop. |
| Loki `/readyz` returns `404` | Loki uses `/ready`; Cerberus uses `/readyz`. Not a health problem. |
| NGINX pod `Running` but `0/1 Ready` | Readiness probe path must be `/ready` (served by NGINX), not `/`. |
| NGINX "buffered to a temporary file" warning | Large upstream response; request still succeeds. Tune proxy buffers if needed. |
| Dashboard panel returns no data through Cerberus | Check the label name: `namespace` (Loki) vs `k8s_namespace_name` (Cerberus). See Issue 1. |
| Nothing in `kubectl logs` for Cerberus | Info-level logging may not record each request; this alone does not indicate failure. |
| Grafana stops showing logs | Check that the tunnel terminal is still running, `curl http://127.0.0.1:32446/ready` on the Grafana VM, and that the NGINX and Cerberus pods are `Running`. |

## 16. Next Steps

1. Fix and re-test NGINX label-values routing (Issue 2).
2. Decide the legacy `namespace` strategy: update dashboard queries, or implement and test a Cerberus alias covering matchers and label discovery (Issue 1).
3. Run the revised count query in the comparison panel and verify its result against ClickHouse using aligned tables and windows.
4. Validate every existing dashboard panel individually and record supported and unsupported queries.
5. Run a real Fluent Bit ingestion test through the router and confirm logs arrive in Loki.
6. Run a controlled parity and latency comparison (same query, same window, repeated runs).
7. Create a read-only ClickHouse user for Cerberus.
8. Replace the laptop tunnel and port-forward dependency with a stable exposure method before any wider rollout.
9. Document the supported-query surface and the rollback path before any cutover decision.

## 17. Current Status

| Area | Status | Evidence / caveat |
|---|---|---|
| Fluent Bit → Loki / Gigapipe / ClickHouse | Configured | Confirmed from ConfigMaps |
| OTel file logs → `otel_logs` | Configured, queryable | ~2.7M rows at last check |
| Cerberus deployment and readiness | Verified | `/readyz` 200, ClickHouse `ok`, schema `ready` |
| Cerberus labels, `query_range`, line filter | Tested successfully | Specific query shapes only |
| NGINX read routing to Cerberus | Verified | Active config, upstream `10.96.165.8:8080` |
| NGINX label-values routing | Issue found | Falls back to Loki; fix pending |
| Grafana `loki-1` datasource | Preserved and working | URL unchanged; logs visible in Explore |
| ClickHouse SQL on `otel_logs` | Verified | `system.query_log` with namespace and time filters |
| Exact request-to-SQL correlation | Not established | No unique request ID |
| Legacy `namespace` label | Known gap | `k8s_namespace_name` works; alias not deployed |
| Push path with real payload | Not established | Empty-payload `204` only |
| Full LogQL / Loki API compatibility | Not established | Selected shapes only |
| Result parity and performance | Not established | Needs controlled test |
| All existing dashboard panels | Pending | Label differences and timeouts observed |

## 18. Conclusion and Recommendation

### 18.1 Verdict

**Partially feasible, with conditions.** Using Cerberus as a Loki-compatible query layer over ClickHouse works for the tested query shapes, and the existing Grafana Loki datasource did not need to change. Full dashboard compatibility is not yet proven, and two open items must be resolved before this can be called a drop-in replacement for Loki queries.

### 18.2 What the POC proved

| Finding | Evidence |
|---|---|
| The existing Grafana datasource `loki-1` can stay unchanged and still return logs from ClickHouse | Logs visible in Explore through `127.0.0.1:32446` for two namespaces |
| Loki queries are translated into ClickHouse SQL | `system.query_log` shows `SELECT Body AS Line ... FROM otel_logs` with namespace `PREWHERE`, time window, `ORDER BY`, `LIMIT` |
| Query and write paths can be separated without touching ingestion | NGINX routes reads to Cerberus and `/loki/api/v1/push` to Loki; Loki Service selector and Fluent Bit config unchanged |
| Cerberus works on the existing `otel_logs` table with no DDL and no data changes | Readiness `ok`/`ready`; labels, `query_range` and line-filter queries succeeded |
| The change is reversible | One tunnel mapping to restore (Section 10.3) |

### 18.3 What is not yet proven

| Gap | Why it matters |
|---|---|
| Legacy `namespace` label (Issue 1) | Existing dashboards that use `{namespace="..."}` can return no data through Cerberus |
| NGINX label-values routing (Issue 2) | `label/<name>/values` requests currently fall back to Loki, so label browsers and variables may not reflect Cerberus |
| All existing dashboard panels | Only selected queries and Explore were tested; no panel-by-panel validation |
| Result parity and latency | Same-query, same-window, repeated-run comparison against Loki has not been done |
| Real ingestion through the router | Push route was only tested with an empty payload |
| Request-to-SQL correlation | Evidence is strong but not one-to-one |

### 18.4 Recommendation

1. **Continue to the next phase, conditionally.** The approach is technically sound for logs and worth pursuing, but it should not yet be presented as a full Loki replacement.
2. **Resolve the two blockers first:** fix NGINX label-values routing (a one-line regex change plus re-test), and choose a strategy for the legacy `namespace` label. Updating dashboard queries to `k8s_namespace_name` is the fastest option; adding a `namespace` alias in Cerberus (matchers and label discovery) preserves existing dashboards unchanged and is closer to the original goal, but needs a code change and a test cycle.
3. **Then validate breadth:** run the real dashboards panel by panel, record supported and unsupported queries, and run a controlled parity and latency comparison.
4. **Treat the current setup as POC-grade only.** Before any wider use, add a read-only ClickHouse user, replace the laptop tunnel and port-forward with a stable exposure method, and consider more than one replica for NGINX and Cerberus.
5. **Do not cut over** the datasource or any Service selector until the supported query surface, known differences and rollback path are documented and signed off.

### 18.5 Production blockers (summary)

- Open label-compatibility gap and label-values routing issue.
- No full dashboard validation or parity/latency data.
- Passwordless, unrestricted ClickHouse `default` user.
- Single replicas and a laptop-dependent tunnel.
- Push path and ingestion not validated with a real payload.

## 19. Appendix: Verification Commands

```bash
# Cerberus health and Loki API (via port-forward)
kubectl port-forward -n cerberus svc/cerberus 8081:8080
curl -i http://127.0.0.1:8081/readyz
curl -i http://127.0.0.1:8081/loki/api/v1/labels
curl -sS http://127.0.0.1:8081/loki/api/v1/label/k8s_namespace_name/values
curl -sS http://127.0.0.1:8081/loki/api/v1/label/namespace/values      # returns empty list (Issue 1)

# Pods and services
kubectl get pods,svc -n cerberus -o wide
kubectl get pods,svc -n log-router -o wide
kubectl get svc loki-external -n traditional-observability -o wide
kubectl get endpoints loki-external -n traditional-observability -o wide

# NGINX
kubectl logs -n log-router deploy/log-router-nginx --tail=50
kubectl exec -n log-router deployment/log-router-nginx -- curl -sS -i http://127.0.0.1:8080/ready

# From the Grafana VM
curl -sS -i --max-time 5 http://127.0.0.1:32446/ready
curl -sS -G --max-time 10 'http://127.0.0.1:32446/loki/api/v1/query_range' \
  --data-urlencode 'query={k8s_namespace_name="traditional-observability"}' \
  --data-urlencode "start=$(date -d '15 minutes ago' +%s)" \
  --data-urlencode "end=$(date +%s)" --data-urlencode 'limit=5'
sudo ss -lntp | grep -E ':(32446|18081)\b'

# Confirm the SQL reached ClickHouse (see Section 11.5 for the full query)
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query \
  "SYSTEM FLUSH LOGS"
```
