# ClickHouse Unified Observability
## 1. Project Overview, Architectural Scope & Environment Specifications
### 1.1 Objective and Foundational Strategy
The primary objective of this Proof of Concept (POC) is to evaluate **ClickHouse as a centralized, high-performance analytical storage backend (OLAP)** capable of unifying the three primary IT observability signals:
* **Logs:** Capturing container stdout and stderr events from Kubernetes workloads, including structured application events.
* **Metrics:** Scraping continuous infrastructure, container, and business-level time-series measurements.
* **Traces:** Ingesting distributed end-to-end transaction contexts across microservices.

In traditional enterprise environments, monitoring typically relies on fragmented architectures with separate storage engines for each telemetry signal (e.g., Elasticsearch or Loki for logs, Prometheus for metrics, and Jaeger or Tempo for traces). This multi-backend model creates operational silos, increases infrastructure costs, and makes troubleshooting difficult because engineers must navigate multiple user interfaces and query syntaxes.

This POC establishes three decoupled, purpose-built ingestion pipelines that converge directly into ClickHouse:
1. **Direct Log Ingestion:** Fluent Bit collects and enriches container logs, transforms records using a custom Lua script, and sends them directly to ClickHouse via HTTP (`JSONEachRow`).
2. **Pull-Based Metrics Collection:** The OpenTelemetry Collector uses its Prometheus Receiver to scrape metrics targets directly without deploying a dedicated Prometheus server, exporting records over ClickHouse's native TCP protocol.
3. **Application Trace Export:** Microservices instrumented with the OpenTelemetry SDK push spans over OTLP/HTTP to the OpenTelemetry Collector, which batches and persists them into ClickHouse trace tables.
4. **Unified Query & Visualization:** Grafana connects to ClickHouse as a centralized SQL datasource, enabling simultaneous cross-signal querying and trace-to-log correlation via shared identifiers (`trace_id`).

### 1.2 POC Scope Matrix
| Dimension | In-Scope Components & Capabilities | Out-of-Scope Production Items |
| :--- | :--- | :--- |
| **Kubernetes Infrastructure** | Local single-node Kind cluster (`v1.37.0`), local volume provisioning. | Multi-node High Availability (HA), multi-cluster federation, cross-AZ failover. |
| **Log Pipeline** | Fluent Bit DaemonSet, CRI/Docker multiline parsing, Kubernetes filter, custom Lua transformation, direct ClickHouse HTTP insert. | OpenTelemetry Collector log ingestion, syslog/journald inputs, enterprise log masking. |
| **Metrics Pipeline** | Node Exporter, cAdvisor (via API server proxy), kube-state-metrics, custom FastAPI `/metrics`, OTel Collector Prometheus Receiver, ClickHouse native exporter. | Dedicated Prometheus Server, Alertmanager rules, Thanos/Cortex integration. |
| **Tracing Pipeline** | FastAPI manual & automatic SDK instrumentation, HTTPX trace context propagation (W3C TraceContext), OTLP/HTTP exporter, ClickHouse trace schema. | Advanced sampling strategies (tail-based sampling), Jaeger/Zipkin standalone UIs. |
| **Storage Layer** | ClickHouse Server (`26.10.4.207`) via Operator, ClickHouse Keeper coordination, dedicated PVCs, MergeTree-family engines. | Multi-shard/multi-replica topologies, backup/restore pipelines, TLS encryption, strict RBAC/user isolation. |
| **Visualization & Correlation** | Grafana (`12.3.1`), ClickHouse HTTP plugin datasource, unified SQL dashboards, cross-signal queries correlating `trace_id`. | Production SSO/SAML integration, enterprise alerting, external ingress/load balancers. |

### 1.3 Expected Outcomes and POC Boundaries
At the conclusion of the POC, the implementation demonstrates that:
* Logs, metrics, and traces can be gathered via independent ingestion pipelines without operational interference.
* ClickHouse can reliably store all three observability signals in a single database using optimized columnar schemas.
* Collected data can be queried efficiently using standard SQL.
* Grafana can visualize logs, metrics, and traces from the unified ClickHouse backend.
* Application logs and distributed traces can be correlated bi-directionally using shared `trace_id` values.
* The architecture provides a valid foundation for evaluating future production-scale observability platforms.

> **POC Limitation:** This implementation is a local functional validation on a single-node Kind cluster, not a production-readiness or high-availability certification exercise.

### 1.4 Architecture of the Three Ingestion Pipelines
#### 1.4.1 Logs Pipeline Architecture
```text
Kubernetes Workloads
        |
        v
Container stdout/stderr
        |
        v
Fluent Bit (DaemonSet)
        |
        v
ClickHouse (HTTP Port 8123)
```
* **Collection:** Fluent Bit runs as a DaemonSet on the Kubernetes node and reads container logs directly from `/var/log/containers/*.log`.
* **Processing:** Fluent Bit enriches each log with Kubernetes metadata (namespace, pod name, container name) and executes a custom Lua transformation script to format the record into 13 standardized columns.
* **Delivery:** Fluent Bit sends the transformed JSON rows directly to ClickHouse over HTTP on port 8123 using the `JSONEachRow` format.
* **Separation:** The OpenTelemetry Collector is not involved in the log ingestion path.

#### 1.4.2 Metrics Pipeline Architecture
```text
Node Exporter (Port 9100)
cAdvisor (API Server Proxy)
kube-state-metrics (Port 8080)
Order API (/metrics Port 8000)
Payment Service (/metrics Port 8001)
        |
        v
OpenTelemetry Collector (Prometheus Receiver)
        |
        v
ClickHouse Exporter (Native TCP Port 9000)
        |
        v
ClickHouse (Database: observability)
```
* **Scraping:** The OpenTelemetry Collector uses its built-in Prometheus Receiver to scrape metrics targets directly at configured intervals (15s for applications, 30s for infrastructure).
* **Processing:** Metrics pass through memory limiter and batch processors.
* **Delivery:** The Collector exports metrics to ClickHouse over native TCP on port 9000, where tables like `otel_metrics_gauge` and `otel_metrics_sum` are automatically created and populated.
* **Separation:** No dedicated Prometheus Server or Alertmanager is deployed.

#### 1.4.3 Distributed Tracing Pipeline Architecture
```text
       Order API (Port 8000)
            |
            | HTTP Call (W3C Trace Context)
            v
   Payment Service (Port 8001)
            |
            | OpenTelemetry SDK
            v
       OTLP HTTP (Port 4318)
            |
            v
 OpenTelemetry Collector (OTLP Receiver)
            |
            v
 ClickHouse Exporter (Native TCP Port 9000)
            |
            v
 ClickHouse (Table: otel_traces)
```
* **Instrumentation:** Both microservices use the OpenTelemetry Python SDK and automatic instrumentation (`FastAPIInstrumentor`, `HTTPXClientInstrumentor`).
* **Transport:** Trace spans are pushed over HTTP to the OpenTelemetry Collector on port 4318 using the standard OTLP protocol.
* **Storage:** The Collector writes spans into `observability.otel_traces` over native TCP on port 9000.

#### 1.4.4 Ingestion Pipeline Summary
| Signal | Collection Component | Processing / Scraping Engine | Storage Destination | Protocol / Port |
| :--- | :--- | :--- | :--- | :--- |
| **Logs** | Fluent Bit (DaemonSet) | Kubernetes filter & Lua transform | ClickHouse (`kubernetes_logs`) | HTTP (`8123`) |
| **Metrics** | Prometheus Exporters & Apps | OTel Collector Prometheus Receiver | ClickHouse (`otel_metrics_*`) | Native TCP (`9000`) |
| **Traces** | OpenTelemetry Python SDK | OTel Collector OTLP HTTP Receiver | ClickHouse (`otel_traces`) | Native TCP (`9000`) |

### 1.5 Technology Stack Specifications
| Layer / Role | Technology Selected | Component Version | Architectural Purpose |
| :--- | :--- | :--- | :--- |
| **Host System** | Ubuntu 22.04.5 LTS | Kernel Linux 5.15+ | Bare-metal host (Dell Latitude 3420, i5-1135G7, 8 vCPUs, 15 GiB RAM, 2 GiB Swap). |
| **Container Engine** | Docker | 29.7.2 | Base container execution engine. |
| **Cluster Orchestration**| Kubernetes (Kind) | Cluster: v1.37.0, Kind: v0.33.0 | Isolated local Kubernetes control-plane environment. |
| **Package Manager** | Helm | v4.3.0 | Deployment packaging for Operator, Collector, Fluent Bit, and Grafana. |
| **Cluster Controller** | ClickHouse Operator | Chart: 0.0.8, Image: v0.0.8 | Declarative CRD reconciliation for ClickHouse and Keeper. |
| **Coordination Engine** | ClickHouse Keeper | Image: latest | Raft-based coordination replacing Apache ZooKeeper. |
| **Analytical Database** | ClickHouse Server | Image: latest (26.10.4.207) | Columnar storage engine for logs, metrics, and traces. |
| **Log Collection Agent** | Fluent Bit | Chart: 0.58.3 | Node DaemonSet for log scraping, filtering, Lua transformation, and HTTP export. |
| **Metrics Exporters** | Node Exporter, KSM, cAdvisor | Standard community releases | Node-level, Kubernetes object-level, and container-level metrics scraping endpoints. |
| **Telemetry Collector** | OpenTelemetry Collector Contrib | Chart: 0.178.3, Image: 0.163.2 | Scrapes Prometheus endpoints, receives OTLP traces, exports natively to ClickHouse. |
| **Microservices** | Python FastAPI / Uvicorn | Python 3.11-slim | Demo apps (`order-api`, `payment-service`) generating HTTP telemetry. |
| **SDK & Tracing** | OpenTelemetry Python SDK | API/SDK 1.x, HTTPX instrumentor | Generates spans, injects W3C headers, exports via OTLP/HTTP. |
| **Visualization Layer** | Grafana | Chart: 10.5.15, App: 12.3.1 | Single-pane SQL dashboarding and cross-signal correlation interface. |

### 1.6 Workspace Structure & Project-Local Tooling
To isolate dependencies without altering system binaries, the POC workspace was structured under `~/clickhouse-unified-poc`:
```text
clickhouse-unified-poc/
├── application/
│   ├── .venv/               # Shared Python 3.11 validation environment
│   ├── order-api/           # Order API service code, requirements, and Dockerfile
│   └── payment-service/     # Payment service code, requirements, and Dockerfile
├── bin/
│   └── kubectl              # Project-local verified kubectl binary (v1.37.1)
├── configs/                 # Helm custom values, Lua transform scripts, and RBAC manifests
├── docs/                    # Technical documentation and notes
├── helm/                    # Archived and pulled Helm chart packages (.tgz)
├── kubernetes/              # Declarative application and service manifests
└── README.md                # Comprehensive proof of concept master documentation
```
Because the host machine possessed an older system client (`kubectl v1.29.15`), a project-local `kubectl v1.37.1` binary was downloaded, verified against official SHA256 checksums (`kubectl: OK`), and installed into `~/clickhouse-unified-poc/bin/kubectl` to match the Kind cluster's API version (`v1.37.0`):

```bash
cd ~/clickhouse-unified-poc
mkdir -p bin

# Download client binary and checksum
curl -LO https://dl.k8s.io/release/v1.37.1/bin/linux/amd64/kubectl
curl -LO https://dl.k8s.io/release/v1.37.1/bin/linux/amd64/kubectl.sha256

# Verify binary integrity
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install binary with execute permissions
install -m 0755 kubectl bin/kubectl
```
Executing `./bin/kubectl version --client` confirmed successful installation:
```text
Client Version: v1.37.1
Kustomize Version: v5.8.1
```
## 2. Phase 2: ClickHouse Installation, Keeper Coordination & Kubernetes Deployment
### 2.1 Rationale for ClickHouse in Observability: OLTP vs. OLAP
Observability workloads generate massive volumes of append-only, time-stamped events. Traditional relational databases (OLTP) store data row by row, which is optimized for transactional record updates and point lookups (e.g., e-commerce order checkout or bank transfers). In contrast, analytical queries over telemetry data typically scan millions of records across only a handful of specific columns—such as aggregating error counts, calculating average request latency, or grouping request counts by service name:

```sql
SELECT
    service_name,
    count() AS total_errors
FROM observability.kubernetes_logs
WHERE level = 'ERROR'
GROUP BY service_name;
```
In a row-oriented database, calculating this aggregate requires loading every single row—including message strings, attributes, and metadata—from disk into memory. ClickHouse, as an Online Analytical Processing (OLAP) columnar database, stores each column in separate, contiguous data files. When computing aggregates or filtering on `service_name` and `level`, ClickHouse reads only those specific columns, eliminating unnecessary disk I/O. Furthermore, storing identical data types contiguously allows ClickHouse to achieve exceptional data compression ratios (via LZ4 and ZSTD), significantly reducing storage footprint for repetitive log and metric strings.

Key ClickHouse features utilized in this architecture include:
* **Vectorized Query Execution:** Processing data chunks using SIMD CPU instructions for high throughput.
* **The MergeTree Engine Family:** Writing immutable parts in background batches, continuously merging them while sorting on primary keys and applying partitioning schemas.
* **Declarative SQL Syntax:** Direct support for analytical primitives (`count()`, `uniq()`, `argMax()`, `groupUniqArray()`, `histogram()`) without requiring proprietary query domain-specific languages.

#### 2.1.1 Columnar Storage Mechanics: A Concrete Example
To visualize how a columnar database optimizes analytics, consider a table of user records:
| ID | Name | City | Age |
| :---: | :--- | :--- | :---: |
| 1 | Rahul | Delhi | 25 |
| 2 | Amit | Mumbai | 28 |
| 3 | Ravi | Noida | 30 |

In traditional row-oriented storage, records are written consecutively on disk:
```text
Row 1: 1, Rahul, Delhi, 25
Row 2: 2, Amit, Mumbai, 28
Row 3: 3, Ravi, Noida, 30
```
In columnar storage, values belonging to each column are stored together in separate files:
```text
ID:   1, 2, 3
Name: Rahul, Amit, Ravi
City: Delhi, Mumbai, Noida
Age:  25, 28, 30
```
When executing an analytical aggregation such as calculating the average age:
```sql
SELECT AVG(Age)
FROM users;
```
* In a row-oriented database, the engine reads every column of every row (ID, Name, City, Age) from disk into memory, causing heavy I/O overhead.
* In a columnar database, the engine reads **only the Age column data file** from disk, completely ignoring the other columns and dramatically reducing disk reads.

#### 2.1.2 Three Telemetry Signals in ClickHouse
Observability systems produce three primary signals, each with distinct data structures:
| Signal | Core Data Characteristics | Representative SQL Query |
| :--- | :--- | :--- |
| **Logs** | High-cardinality event strings, error levels | `SELECT count() FROM observability.kubernetes_logs;` |
| **Metrics** | Numerical measurements over time | `SELECT count() FROM observability.otel_metrics_gauge;` |
| **Traces** | Distributed call graphs, span durations | `SELECT count() FROM observability.otel_traces;` |

### 2.2 ClickHouse Operator Architecture & CRD Mechanics
To avoid the operational fragility of manually managing Kubernetes StatefulSets, headless services, and storage volume bindings, the POC deploys the official ClickHouse Operator. The Operator runs as a controller manager within the `clickhouse-operator-system` namespace and registers two core Custom Resource Definitions (CRDs):
* `clickhouseclusters.clickhouse.com`: Defines the target ClickHouse server topology, including shards, replicas, resource allocations, persistent volume claim templates, and keeper connection configurations.
* `keeperclusters.clickhouse.com`: Defines the coordination cluster topology, storage claims, and pod specifications for ClickHouse Keeper.

The reconciliation cycle operates continuously:
```text
          Kubernetes API Server
                   |
                   v
     Custom Resource (ClickHouseCluster)
                   |
                   v
       ClickHouse Operator Controller
                   |
      +------------+------------+
      |            |            |
      v            v            v
  StatefulSet   Services       PVCs
      |            |            |
      +------------+------------+
                   |
                   v
         ClickHouse Server Pod
```
### 2.3 ClickHouse Keeper & Distributed Coordination
ClickHouse Keeper is a C++ implementation of the ZooKeeper-compatible coordination protocol that maintains cluster consensus, tracks replica states, coordinates data part merging, and guarantees distributed DDL execution. 

In this POC, ClickHouse Keeper was deployed as an independent Kubernetes Custom Resource (`KeeperCluster`) separate from the ClickHouse server. This decoupling ensures:
* **Fault Domain Separation:** Coordination state is isolated from analytical query processing.
* **Dedicated Storage Persistence:** The ClickHouse database and Keeper coordination state use separate PersistentVolumeClaims. ClickHouse database operations cannot starve Keeper of disk I/O, preventing consensus timeouts.

```text
        ClickHouse Cluster (observability)
                       |
        +--------------+--------------+
        |                             |
        v                             v
 ClickHouse Server Pod         ClickHouse Keeper Pod
(clickhouse-clickhouse-0-0-0) (clickhouse-keeper-keeper-0-0)
        |                             |
        v                             v
  ClickHouse PVC (5Gi)           Keeper PVC (2Gi)
```
#### 2.3.1 Keeper Cluster Coordination Role
In distributed ClickHouse deployments, replicas coordinate their activities through Keeper:
```text
        ClickHouse Cluster
                |
        +-------+-------+
        |               |
        v               v
    Replica 1        Replica 2
        |               |
        +-------+-------+
                |
                v
        ClickHouse Keeper
```
#### 2.3.2 Storage Volume Decoupling Rationale
ClickHouse Server and ClickHouse Keeper use separate Persistent Volume Claims:
| Component | Storage Size | Storage Class | Stored Data Types |
| :--- | :--- | :--- | :--- |
| **ClickHouse Server** | `5Gi` | `standard` | Database tables, columnar part files, marks, analytical metadata |
| **ClickHouse Keeper** | `2Gi` | `standard` | Raft consensus logs, snapshots, cluster coordination state |

Decoupling storage ensures that heavy analytical queries or large batch log inserts never consume Keeper's disk I/O, preventing coordination timeouts and consensus loss.

### 2.4 Declarative Deployment Configurations
#### 2.4.1 Prerequisite: cert-manager Installation
Before installing the ClickHouse Operator, `cert-manager` was deployed into the cluster to handle internal certificate provisioning and admission webhook validation required by the operator controller.

#### 2.4.2 ClickHouse Operator Helm Deployment
Installed into the `clickhouse-operator-system` namespace using official chart `clickhouse-operator-helm` version `0.0.8`:
```bash
helm upgrade --install clickhouse-operator \
  clickhouse-operator-helm \
  --namespace clickhouse-operator-system \
  --version 0.0.8
```
#### 2.4.3 ClickHouse Keeper Manifest Specification
Configured with dedicated resource reservations and independent persistence:
| Parameter | Configured Value | Specification Rationale |
| :--- | :--- | :--- |
| **Resource Name** | `clickhouse-keeper` | Declarative identifier for `KeeperCluster` CR |
| **Namespace** | `clickhouse` | Dedicated namespace for persistence infrastructure |
| **Replicas** | `1` | Functional single-node quorum for local POC |
| **Image** | `docker.io/clickhouse/clickhouse-keeper:latest` | Official Keeper container runtime |
| **CPU Requests / Limits** | `100m` / `500m` | Guarantees minimum compute for Raft tick processing |
| **Memory Requests / Limits** | `256Mi` / `512Mi` | Sufficient for coordination transaction logs |
| **Storage Capacity** | `2Gi` (`standard` StorageClass) | ReadWriteOnce persistent disk for coordination snapshots |
| **Cluster Domain** | `cluster.local` | Standard Kubernetes internal DNS suffix |

#### 2.4.4 ClickHouse Server Cluster Manifest Specification
Configured via `ClickHouseCluster` Custom Resource:
| Parameter | Configured Value | Specification Rationale |
| :--- | :--- | :--- |
| **Cluster Name** | `clickhouse` | Cluster identifier across services |
| **Namespace** | `clickhouse` | Shared infrastructure namespace with Keeper |
| **Topology** | Shards: `1`, Replicas: `1` | Single shard and single replica for local validation |
| **Image Tag** | `docker.io/clickhouse/clickhouse-server:latest` | Evaluated server version `26.10.4.207` |
| **CPU Requests / Limits** | `250m` / `1000m` (1 Core) | Allocates compute headroom for analytical aggregations |
| **Memory Requests / Limits**| `512Mi` / `1024Mi` (1 GiB) | Guards Kind node against memory exhaustion |
| **Storage Capacity** | `5Gi` (`standard` StorageClass) | Persistent storage for data parts and metadata |
| **Keeper Reference** | `clickhouse-keeper` | Connects server directly to deployed Keeper CR |
| **Database Sync** | `Enabled` | Automates database metadata replication |
| **Security / TLS** | `Disabled` | Kept unencrypted for local functional isolation |

```text
              ClickHouse Cluster
                      |
             +--------+--------+
             |                 |
             v                 v
          Shard 1           Shard 2
             |                 |
       +-----+-----+     +-----+-----+
       |           |     |           |
       v           v     v           v
    Replica 1  Replica 2 Replica 1  Replica 2
```
In this POC, the deployment is configured with `shards: 1` and `replicas: 1`. This single-node setup is suitable for functional validation, while production deployments require multi-shard and multi-replica topologies.

### 2.5 Kubernetes Networking, Services and Endpoints Mapping
The Operator instantiated headless services to provide stable DNS endpoints for StatefulSet pods:
| Service Name | Service Type | Port Mappings | Functional Role |
| :--- | :--- | :--- | :--- |
| `clickhouse-clickhouse-headless` | Headless (`ClusterIP: None`) | `8123/TCP` (HTTP), `9000/TCP` (Native TCP), `9363/TCP` (Metrics) | Client queries, OpenTelemetry Collector TCP export, Fluent Bit HTTP inserts, Grafana data source |
| `clickhouse-clickhouse-internal-0-0` | Headless (`ClusterIP: None`) | `9001/TCP` (Replica TCP), `9002/TCP` (Internal Interserver), `9009/TCP` (Data Sync) | Replica communication, interserver data exchange |
| `clickhouse-keeper-keeper-headless` | Headless (`ClusterIP: None`) | `2181/TCP` (ZooKeeper client), `9234/TCP` (Raft consensus), `9090/TCP` (Metrics) | ClickHouse-to-Keeper coordination and quorum election |

Endpoint mapping confirmed healthy association with container IPs: `clickhouse-clickhouse-headless` (`10.244.0.14:8123, 9000, 9363`), `clickhouse-clickhouse-internal-0-0` (`10.244.0.14:9001, 9002, 9009`), and `clickhouse-keeper-keeper-headless` (`10.244.0.11:2181, 9090, 9234`).

### 2.6 Deployment Verification, Connectivity & DNS Investigation
#### 2.6.1 Cluster Readiness and Client Validation
Cluster status was verified via custom resource inspection:
```bash
kubectl get clickhouseclusters.clickhouse.com -A
```
*Output confirmed:* `READY: True`, `STATUS: All shards are ready`, `READYREPLICAS: 1/1`, `VERSION: 26.10.4.207`.

Client connectivity was validated directly within the server pod using `clickhouse-client`:
```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client --query "SELECT 1;"
```
*Output returned:* `1` (Elapsed: `0.001 sec.`), confirming database responsiveness.

#### 2.6.2 Operator Reconciliation & DNS Timeout Diagnostic
During initial cluster initialization, the ClickHouse Operator controller logged replica probing errors:
```text
failed to probe replica
dial tcp: lookup clickhouse-clickhouse-internal-0-0.clickhouse.svc.cluster.local: i/o timeout
```
A technical investigation of the Kubernetes DNS subsystem was conducted:
1. Checked Operator pod DNS policy: Configured as `dnsPolicy: ClusterFirst` with default CoreDNS nameserver (`10.96.0.10`) and `options ndots:5`.
2. Observed external search domains inherited from the host environment: `netbird.selfhosted` and `tail5caa65.ts.net`.
3. Verified container runtime security: Ephemeral debug containers using BusyBox were rejected by the admission controller because the Operator pod enforced strict `runAsNonRoot` policies.
4. Final Reconciliation: Without modifying core DNS infrastructure, the cluster completed service endpoint registration, resolved the internal headless hostname, and transitioned to `READY: True` with all shards healthy.

### 2.7 Phase 2 Deployment Summary
| Component / Step | Namespace | Observed Result |
| :--- | :--- | :--- |
| **Kind Cluster** | Host Engine | Running single-node cluster (v1.37.0) |
| **cert-manager** | `cert-manager` | Installed and operational |
| **ClickHouse Operator** | `clickhouse-operator-system` | Running with zero restarts |
| **Operator CRDs** | Kubernetes API | `clickhouseclusters` & `keeperclusters` registered |
| **ClickHouse Keeper** | `clickhouse` | Pod ready, Raft coordination active |
| **ClickHouse Server** | `clickhouse` | Running version 26.10.4.207 |
| **Persistent Storage** | `clickhouse` | 5Gi (ClickHouse) & 2Gi (Keeper) PVCs Bound |
| **Kubernetes Services** | `clickhouse` | Headless services available on ports 8123, 9000, 9363 |
| **Client SQL Access** | `clickhouse` | Native client connected, SELECT 1 verified |

## 3. Phase 3: Observability Schema Preparation & Demo Microservices Development
### 3.1 Observability Database Initialization & Multi-Signal Schema Strategy
A dedicated database named `observability` was established inside ClickHouse:
```sql
CREATE DATABASE IF NOT EXISTS observability;
```
#### Why Separate Tables for Different Signals?
Observability signals exhibit fundamentally divergent data access patterns, retention requirements, and structural characteristics:
* **Logs (`kubernetes_logs`):** High-cardinality text payloads with structured metadata (service, namespace, severity, trace IDs). Written as independent event records; queried via text filters, regular expressions, and time-range scans.
* **Metrics (`otel_metrics_*`):** Low-cardinality numerical measurements recorded at regular intervals. Requires time-series downsampling, bucket aggregations, and distinct representations for gauges, cumulative counters, and distribution histograms.
* **Traces (`otel_traces`):** Relational acyclic graphs linking parent and child operations. Spans require nanosecond-precision timestamps, duration computations, and parent span lookups across microservice boundaries.

Separating these signals into dedicated tables guarantees that queries over metric counters do not scan bulky log payloads, while trace trees can be queried efficiently without interference from high-frequency container events.

### 3.2 Microservices Architecture: Inter-Service HTTP Communication
To generate realistic application-layer telemetry, two interconnected microservices were developed using Python FastAPI:
1. **Order API (`order-api`):** The client-facing edge service listening on port `8000`.
2. **Payment Service (`payment-service`):** The downstream backend dependency listening on port `8001`.

```text
                  HTTP Client / Swagger UI
                             |
                             v
                 Order API (Port 8000)
                 Namespace: applications
                             |
                   HTTP Request (HTTPX)
                   W3C TraceContext Header
                             v
               Payment Service (Port 8001)
                 Namespace: applications
                             |
           Conditional Logic: Amount >= 10,000?
             /                               \
    Yes (>= 10,000)                      No (< 10,000)
           |                                   |
           v                                   v
    HTTP 400 Declined                   HTTP 200 Success
  PAYMENT_DECLINED log                 Payment processed log
           |                                   |
           +-----------------+-----------------+
                             |
                             v
               Order API Response to Client
             (HTTP 502 Error / HTTP 200 Success)
```
Both services reside in the `applications` namespace. The Order API communicates with Payment Service via Kubernetes internal DNS using the short service name:
```python
PAYMENT_SERVICE_URL = os.getenv(
    "PAYMENT_SERVICE_URL",
    "http://payment-service:8001/payments"
)
```
```text
                 Client
                   |
                   v
              Order API (Port 8000)
                   |
            HTTP Request (POST /payments)
                   v
            Payment Service (Port 8001)
                   |
             Payment Result (HTTP 200 / 400)
                   v
              Order API
                   |
             Client Response (HTTP 200 / 502)
```
#### 3.2.1 Component Responsibilities
* **Order API:** Accepts order creation requests, generates unique UUID order identifiers, stores order details in-memory, initiates HTTP calls to Payment Service, handles payment responses, generates structured JSON logs, creates OpenTelemetry spans, and exposes `/metrics`.
* **Payment Service:** Accepts payment processing requests, evaluates amount conditions, processes transactions or rejects them, generates structured JSON logs, creates OpenTelemetry spans, and exposes `/metrics`.

### 3.3 Microservice Implementation Specifications
#### 3.3.1 Order API Service
* **Directory:** `application/order-api/`
* **Endpoints:**
  * `GET /health`: Health check probe returning service status.
  * `POST /orders`: Accepts order payload (`customer_id`, `amount`), generates unique UUID `order_id`, and dispatches payment processing request to Payment Service.
  * `GET /orders/{order_id}`: Fetches order state from in-memory cache.
  * `POST /simulate-error`: Simulates an uncaught internal exception for error pipeline testing.
  * `GET /metrics`: Exposes Prometheus-compatible application counters and histograms.

#### 3.3.2 Payment Service & Deterministic Failure Injection
* **Directory:** `application/payment-service/`
* **Endpoints:**
  * `GET /health`: Returns service health status.
  * `POST /payments`: Processes incoming payment requests.
  * `POST /simulate-error`: Generates test errors.
  * `GET /metrics`: Exposes Prometheus metrics.
* **Deterministic Failure Logic:**
  To generate reliable test datasets representing both nominal and degraded states, the Payment Service evaluates transaction amounts:
  ```python
  if request.amount >= 10000:
      payments_processed.labels(status="failed").inc()
      log_event(
          "ERROR",
          "Payment processing failed",
          payment_id=payment_id,
          order_id=request.order_id,
          amount=request.amount,
          error_type="PAYMENT_DECLINED"
      )
      raise HTTPException(
          status_code=400,
          detail="Payment declined: simulated failure"
      )
  ```
  Transactions below 10,000 succeed (HTTP 200); transactions at or above 10,000 trigger a simulated decline (HTTP 400), causing Order API to catch the client error, log an error, and return HTTP 502 Bad Gateway to the caller.

#### 3.3.1 Service Endpoints & Business Logic Matrix
**Order API Endpoints:**
| HTTP Method | Route | Operational Function |
| :--- | :--- | :--- |
| `GET` | `/health` | Microservice liveness and readiness probe |
| `POST` | `/orders` | Accepts order payload and invokes Payment Service |
| `GET` | `/orders/{order_id}` | Retrieves order state from in-memory dictionary |
| `POST` | `/simulate-error` | Explicitly generates an internal test exception |
| `GET` | `/metrics` | Exposes Prometheus-compatible application counters and histograms |

**Payment Service Endpoints:**
| HTTP Method | Route | Operational Function |
| :--- | :--- | :--- |
| `GET` | `/health` | Microservice liveness and readiness probe |
| `POST` | `/payments` | Evaluates transaction amount and processes payment |
| `POST` | `/simulate-error` | Explicitly generates a simulated processing error |
| `GET` | `/metrics` | Exposes Prometheus-compatible application counters and histograms |

**Payment Simulation Logic:**
| Transaction Amount | Simulated Decision | Emitted Log Level | HTTP Status Code | Result Details |
| :--- | :--- | :--- | :--- | :--- |
| `amount < 10000` | Approved | `INFO` | `HTTP 200 OK` | `payment_status: SUCCESS` |
| `amount >= 10000` | Declined | `ERROR` | `HTTP 400 Bad Request` | `error_type: PAYMENT_DECLINED` |

### 3.4 Python Dependencies & Runtime Environment
Both services share an identical OpenTelemetry and Prometheus instrumentation foundation.
* **Order API Dependencies (`requirements.txt`):**
  ```text
  fastapi
  uvicorn[standard]
  httpx
  opentelemetry-api
  opentelemetry-sdk
  opentelemetry-instrumentation-fastapi
  opentelemetry-instrumentation-httpx
  opentelemetry-exporter-otlp
  prometheus-client
  ```
* **Payment Service Dependencies (`requirements.txt`):**
  ```text
  fastapi
  uvicorn[standard]
  opentelemetry-api
  opentelemetry-sdk
  opentelemetry-instrumentation-fastapi
  opentelemetry-exporter-otlp
  prometheus-client
  ```
Local package imports and dependency compatibility were validated within a shared virtual environment (`application/.venv/`) before building container images.

### 3.5 Structured JSON Logging Implementation
Rather than emitting unformatted text strings, both microservices emit structured JSON to `stdout`. Every log event automatically captures contextual metadata and distributed trace correlation keys:
| JSON Field | Data Type | Description |
| :--- | :--- | :--- |
| `timestamp` | String (ISO 8601 UTC) | Event generation timestamp with microsecond precision |
| `level` | String (`INFO`, `ERROR`) | Severity level used for filtering and alerting |
| `service_name` | String | Application identifier (`order-api` or `payment-service`) |
| `message` | String | Human-readable log narrative |
| `trace_id` | String (Hex 32-char) | OpenTelemetry trace identifier active during the execution context |
| `span_id` | String (Hex 16-char) | Current active span identifier |
| Dynamic fields | String / Float / UUID | Context-specific attributes (`order_id`, `customer_id`, `amount`, `error_type`) |

#### Verified Production Log Samples
* **Order API Creation Log (Nominal Transaction):**
  ```json
  {
    "timestamp": "2026-11-22T16:43:36.964703+00:00",
    "level": "INFO",
    "service_name": "order-api",
    "message": "Order created successfully",
    "trace_id": "19f538ceb85f2ab83b81df829f2f6dca",
    "span_id": "e7746ae86dcf3a3f",
    "order_id": "1bc448f6-5488-4e09-a97b-9e52b0b7e0c8",
    "customer_id": "CUST-2026-202",
    "amount": 15000.0
  }
  ```
* **Payment Service Error Log (Simulated Failure Transaction):**
  ```json
  {
    "timestamp": "2026-11-22T16:43:36.986756+00:00",
    "level": "ERROR",
    "service_name": "payment-service",
    "message": "Payment processing failed",
    "trace_id": "19f538ceb85f2ab83b81df829f2f6dca",
    "span_id": "0d93d73328f4d2a1",
    "payment_id": "2846bf05-3f47-468f-b45d-e50adbf985fe",
    "order_id": "1bc448f6-5488-4e09-a97b-9e52b0b7e0c8",
    "amount": 15000.0,
    "error_type": "PAYMENT_DECLINED"
  }
  ```
### 3.6 OpenTelemetry SDK Instrumentation & Trace Context Propagation
To enable distributed tracing across service boundaries, both applications configure the OpenTelemetry Python SDK:
1. **TracerProvider Initialization:** Associates the execution runtime with an OpenTelemetry `Resource` specifying `service.name`.
2. **FastAPI Auto-Instrumentation:** `FastAPIInstrumentor.instrument_app(app)` intercepts incoming HTTP requests, extracts incoming traceparent headers if present, and creates a root server span.
3. **HTTPX Client Auto-Instrumentation:** In `order-api`, `HTTPXClientInstrumentor().instrument()` intercepts outgoing HTTP calls to `payment-service`. It automatically injects the W3C TraceContext headers (`traceparent`, `tracestate`) into the outbound HTTP request headers.
4. **Context Extraction:** When `payment-service` receives the HTTP request, its FastAPI instrumentor reads the `traceparent` header, adopting the caller's `trace_id` and registering the caller's client span ID as its `parent_span_id`.

### 3.7 Prometheus Metrics Endpoints
Both microservices initialize custom Prometheus metrics exposed on `/metrics`:
* **Order API:**
  * `orders_created_total` (Counter): Measures cumulative orders submitted.
  * `order_request_duration_seconds` (Histogram): Measures order processing latency distribution.
* **Payment Service:**
  * `payments_processed_total` (Counter, labeled by `status="success"` or `status="failed"`): Tracks cumulative payment transactions.
  * `payment_request_duration_seconds` (Histogram): Tracks payment execution duration.

#### 3.8.1 Dockerfile Specifications
Both applications were containerized using lightweight `python:3.11-slim` images:

```dockerfile
# Order API Dockerfile (application/order-api/Dockerfile)
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
```dockerfile
# Payment Service Dockerfile (application/payment-service/Dockerfile)
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
EXPOSE 8001
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
```
Images were built locally (`order-api:poc-v1`, ~307 MB; `payment-service:poc-v1`, ~305 MB) and loaded directly into the Kind cluster control-plane node:
```bash
kind load docker-image order-api:poc-v1 --name clickhouse-unified-poc
kind load docker-image payment-service:poc-v1 --name clickhouse-unified-poc
```
### 3.8 Containerization & Kubernetes Deployment
#### 3.8.1 Docker Packaging
Both applications use multi-stage-clean patterns based on `python:3.11-slim`:
* `order-api`: Exposes port `8000`, runs `uvicorn main:app --host 0.0.0.0 --port 8000`, tagged as `order-api:poc-v1` (~307 MB).
* `payment-service`: Exposes port `8001`, runs `uvicorn main:app --host 0.0.0.0 --port 8001`, tagged as `payment-service:poc-v1` (~305 MB).

#### 3.8.2 Image Ingestion into Kind
Because Kind nodes operate within an isolated Docker-in-Docker container, locally built images were loaded directly into the cluster:
```bash
kind load docker-image order-api:poc-v1 --name clickhouse-unified-poc
kind load docker-image payment-service:poc-v1 --name clickhouse-unified-poc
```
#### 3.8.3 Declarative Kubernetes Manifests
The microservices were deployed into the `applications` namespace as Kubernetes Deployments and ClusterIP Services:
* `order-api`: Exposes ClusterIP `8000/TCP`, targets container port `8000`, mounts `PAYMENT_SERVICE_URL`.
* `payment-service`: Exposes ClusterIP `8001/TCP`, targets container port `8001`.

During initial deployment, Payment Service temporarily entered `ImagePullBackOff` when the cluster attempted to pull `docker.io/library/payment-service:poc-v1` from Docker Hub. Re-executing `kind load docker-image` populated the Kind node's container image cache, resolving the issue.

### 3.9 Functional End-to-End Testing & Trace Propagation Proof
Local access to the microservices was established via port forwarding:
* Terminal 1: `kubectl port-forward -n applications svc/order-api 8000:8000` -> `http://localhost:8000/docs`
* Terminal 2: `kubectl port-forward -n applications svc/payment-service 8001:8001` -> `http://localhost:8001/docs`

Internal DNS connectivity was validated from within the `order-api` container:
```bash
kubectl exec -it deployment/order-api -n applications -- \
  python -c "import urllib.request; print(urllib.request.urlopen('http://payment-service:8001/health').read().decode())"
```
*Output returned:* `{"status": "healthy", "service": "payment-service"}`.

#### Test Execution & Trace ID Correlation
Two transaction scenarios were executed via Swagger UI:
1. **Scenario 1 (Successful Order):** `{"customer_id": "CUST-2026-201", "amount": 750}` -> Returned `HTTP 200 OK`, `payment_status: "SUCCESS"`.
2. **Scenario 2 (Failed Order):** `{"customer_id": "CUST-2026-202", "amount": 15000}` -> Returned `HTTP 502 Bad Gateway` from `order-api` due to `HTTP 400 Bad Request` from `payment-service`.

Inspecting `kubectl logs` across both pods confirmed that the exact same distributed trace ID was preserved across service boundaries: **Shared Trace ID** `19f538ceb85f2ab83b81df829f2f6dca`, with `order-api` span `e7746ae86dcf3a3f` and `payment-service` span `0d93d73328f4d2a1`.

This confirmed that distributed trace context propagation was operational prior to configuring database export pipelines.

### 3.10 Phase 3 Validation Summary
| Validation Dimension | Verification Mechanism | Observed Result |
| :--- | :--- | :--- |
| **Observability Database** | `CREATE DATABASE IF NOT EXISTS observability;` | Completed |
| **Microservices Development** | FastAPI apps (`order-api`, `payment-service`) | Completed |
| **Python Dependencies** | Shared `.venv` importing FastAPI, OTel, Prometheus | Verified |
| **Container Images** | Docker build & `kind load` | Images active in Kind |
| **Kubernetes Deployment** | Deployments & ClusterIP Services in `applications` | 1/1 Running |
| **Internal DNS Resolution** | Order API invoking Payment Service health check | `HTTP 200 healthy` |
| **Scenario 1 (Success)** | Order creation with `amount = 500` | `HTTP 200 SUCCESS` |
| **Scenario 2 (Failure)** | Order creation with `amount = 15000` | `HTTP 502 / 400 DECLINED` |
| **Structured JSON Logs** | Pod stdout logs containing timestamp, level, trace_id | Verified |
| **Trace Context Propagation** | Same Trace ID (`19f538ceb85f2ab83b81df829f2f6dca`) across apps | Verified |

## 4. Phase 4: Fluent Bit Integration & Direct ClickHouse Log Ingestion
### 4.1 Log Pipeline Architecture & Design Rationale
Rather than funneling bulk container logs through the OpenTelemetry Collector, this architecture assigns log aggregation to **Fluent Bit** deployed as a Kubernetes DaemonSet. Fluent Bit reads node log files directly, enriches records with pod metadata, executes in-flight data transformation via an embedded Lua engine, and streams records straight into ClickHouse over HTTP:

```text
       Kubernetes Node (/var/log/containers/*.log)
                           |
                           v
              Fluent Bit Agent (DaemonSet)
                           |
                     [INPUT: tail]
                (Multiline CRI/Docker)
                           |
                           v
                  [FILTER: kubernetes]
          (Enriches namespace, pod, container)
                           |
                           v
                     [FILTER: lua]
              (clickhouse_transform.lua)
       (Normalizes level, extracts trace_id,
        classifies log_type, formats timestamp)
                           |
                           v
                    [OUTPUT: http]
         (HTTP POST /?query=INSERT... JSONEachRow)
                           |
                           v
       ClickHouse Headless Service (Port 8123)
                           |
                           v
              ClickHouse MergeTree Table
            (observability.kubernetes_logs)
```
Direct HTTP ingestion from Fluent Bit offers distinct architectural advantages:
* **Collector Decoupling:** Isolates the high-volume log pipeline from metrics and tracing pipelines, preventing log surges from degrading metric scraping intervals.
* **Low Memory Footprint:** Fluent Bit's lightweight C runtime consumes minimal node overhead compared to running collector-based log processors on every host.
* **Direct High-Throughput Ingestion:** Leverages ClickHouse's native `JSONEachRow` HTTP streaming endpoint, avoiding intermediate message queue complexity during POC validation.

```text
        Kubernetes Workloads
                 |
                 v
        Container stdout/stderr
                 |
                 v
          Fluent Bit Agent (DaemonSet)
                 |
                 v
          Tail Input Plugin (/var/log/containers/*.log)
                 |
                 v
       Kubernetes Filter Plugin (Metadata Enrichment)
                 |
                 v
          Custom Lua Transformation (clickhouse_transform.lua)
                 |
                 v
          HTTP Output Plugin (Port 8123)
                 |
                 v
             ClickHouse (observability.kubernetes_logs)
```
### 4.2 ClickHouse Log Table DDL & Schema Optimizations
Logs are stored in a dedicated `kubernetes_logs` table structured to accommodate both structured application payloads and unstructured system messages:

```sql
CREATE TABLE observability.kubernetes_logs
(
    timestamp DateTime64(3, 'UTC'),
    cluster_name LowCardinality(String) DEFAULT 'clickhouse-unified-poc',
    namespace LowCardinality(String) DEFAULT 'unknown',
    pod_name String DEFAULT 'unknown',
    container_name LowCardinality(String) DEFAULT 'unknown',
    level LowCardinality(String) DEFAULT 'INFO',
    service_name LowCardinality(String) DEFAULT 'unknown',
    message String DEFAULT '',
    trace_id String DEFAULT '',
    span_id String DEFAULT '',
    attributes String DEFAULT '{}',
    log_type LowCardinality(String) DEFAULT 'application',
    raw_log String DEFAULT ''
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(timestamp)
ORDER BY (namespace, pod_name, timestamp);
```
#### Schema Design Decisions
1. **LowCardinality Dictionary Encoding:** Applied to `cluster_name`, `namespace`, `container_name`, `level`, `service_name`, and `log_type`. Columns with limited sets of distinct strings are internally converted into integer dictionary keys, dramatically accelerating filtering and `GROUP BY` operations while reducing storage size.
2. **Monthly Partitioning (`PARTITION BY toYYYYMM(timestamp)`):** Separates data parts into monthly directories, allowing historical log data to be queried efficiently or pruned instantly via `ALTER TABLE ... DROP PARTITION` without scanning active tables.
3. **Primary Sorting Key (`ORDER BY (namespace, pod_name, timestamp)`):** Physically sorts data parts on disk first by `namespace`, second by `pod_name`, and finally by `timestamp`. This layout provides sub-millisecond seek times for standard operational queries filtering within a specific namespace or workload over a designated time window.
4. **Hybrid Structure Strategy:** Essential operational attributes (`level`, `service_name`, `trace_id`, `span_id`) are hoisted into first-class columns for rapid index seeks, while the complete original payload is preserved intact in `raw_log` for diagnostic fidelity.

### 4.3 Fluent Bit Helm Configuration & DaemonSet Packaging
Fluent Bit was installed via Helm into the `logging` namespace using official chart version `0.58.3`:
```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm pull fluent/fluent-bit --version 0.58.3 --destination ~/clickhouse-unified-poc/helm
```
Configuration settings were maintained in `configs/fluent-bit-values.yaml`, validated via `helm lint`, and deployed with release timeout enforcement.

### 4.4 Ingestion Pipeline Stage Configurations
#### 4.4.1 Input Stage: Tail Plugin
Reads container log files from the node filesystem:
```ini
[INPUT]
    Name tail
    Path /var/log/containers/*.log
    multiline.parser docker, cri
    Tag kube.*
    Mem_Buf_Limit 20MB
    Skip_Long_Lines On
```
* `multiline.parser docker, cri`: Reassembles split multiline stack traces across Docker and CRI (containerd) log drivers.
* `Mem_Buf_Limit 20MB`: Prevents memory exhaustion by applying backpressure to the input tail if output pipelines stall.

#### 4.4.2 Filter Stage: Kubernetes Metadata Enrichment
Enriches raw container records with cluster context:
```ini
[FILTER]
    Name kubernetes
    Match kube.*
    Kube_Tag_Prefix kube.var.log.containers.
    Merge_Log On
    Keep_Log On
    K8S-Logging.Parser On
    K8S-Logging.Exclude On
```
* `Merge_Log On`: Automatically unpacks structured JSON strings emitted inside application containers into first-class Fluent Bit map attributes.
* `Keep_Log On`: Preserves the unparsed raw log string under the `log` attribute.

#### 4.2.1 Schema Column Specifications
| Column Name | Data Type | Default | Operational Purpose |
| :--- | :--- | :--- | :--- |
| `timestamp` | `DateTime64(3, 'UTC')` | — | Millisecond-precision UTC event timestamp |
| `cluster_name` | `LowCardinality(String)` | `'clickhouse-unified-poc'` | Target Kubernetes cluster identifier |
| `namespace` | `LowCardinality(String)` | `'unknown'` | Kubernetes namespace where pod executes |
| `pod_name` | `String` | `'unknown'` | Exact pod name emitting the container log |
| `container_name` | `LowCardinality(String)` | `'unknown'` | Container name within the pod specification |
| `level` | `LowCardinality(String)` | `'INFO'` | Normalized uppercase severity (`INFO`, `ERROR`, `UNKNOWN`) |
| `service_name` | `LowCardinality(String)` | `'unknown'` | Extracted application name or container name |
| `message` | `String` | `''` | Primary human-readable event message |
| `trace_id` | `String` | `''` | 32-character hex W3C distributed trace identifier |
| `span_id` | `String` | `''` | 16-character hex distributed span identifier |
| `attributes` | `String` | `'{}'` | JSON representation of dynamic metadata |
| `log_type` | `LowCardinality(String)` | `'application'` | Category (`application`, `access`, `infrastructure`) |
| `raw_log` | `String` | `''` | Unmodified original log payload from container stdout |

#### 4.4.3 Custom Lua Transformation Filter
To map heterogeneous Kubernetes records cleanly into the 13-column ClickHouse table schema, a custom Lua script was mounted into `/fluent-bit/scripts/clickhouse_transform.lua`:

```lua
function transform_log(tag, timestamp, record)
    local k8s = record["kubernetes"] or {}

    local namespace = k8s["namespace_name"] or "unknown"
    local pod_name = k8s["pod_name"] or "unknown"
    local container_name = k8s["container_name"] or "unknown"

    local raw_log = record["log"] or ""

    if type(raw_log) ~= "string" then
        raw_log = tostring(raw_log)
    end

    local level = record["level"] or "UNKNOWN"
    local service_name = record["service_name"] or container_name
    local message = record["message"] or raw_log

    local trace_id = record["trace_id"] or ""
    local span_id = record["span_id"] or ""

    level = string.upper(tostring(level))

    local log_type = "infrastructure"

    if record["service_name"] ~= nil then
        log_type = "application"
    elseif string.match(raw_log, '"%s*[A-Z]+%s+[^"]*HTTP/') then
        log_type = "access"
    end

    local seconds = math.floor(timestamp)
    local milliseconds = math.floor((timestamp - seconds) * 1000)

    local formatted_time = os.date("!%Y-%m-%d %H:%M:%S", seconds)
        .. string.format(".%03d", milliseconds)

    local output = {
        timestamp = formatted_time,
        cluster_name = "clickhouse-unified-poc",
        namespace = namespace,
        pod_name = pod_name,
        container_name = container_name,
        level = level,
        service_name = service_name,
        message = tostring(message),
        trace_id = tostring(trace_id),
        span_id = tostring(span_id),
        attributes = "{}",
        log_type = log_type,
        raw_log = raw_log
    }

    return 2, timestamp, output
end
```
##### Lua Execution Mechanics
1. **Metadata Fallbacks:** Reads Kubernetes metadata; defaults unassigned fields to `"unknown"`.
2. **Signal Extraction:** Pulls application fields merged by the Kubernetes filter (`service_name`, `level`, `trace_id`, `span_id`).
3. **Severity Normalization:** Forces severity strings to uppercase (`INFO`, `ERROR`) to prevent fragmented aggregations caused by mixed casing (`info` vs `INFO`).
4. **Workload Classification:** Classifies events into `application` (if `service_name` is present), `access` (matching HTTP access patterns), or `infrastructure`.
5. **Timestamp Formatting:** Converts the epoch float timestamp into a UTC string formatted as `YYYY-MM-DD HH:MM:SS.mmm` to match ClickHouse's `DateTime64(3, 'UTC')` parser.
6. **Return Code 2:** Returning `2, timestamp, output` instructs Fluent Bit to replace the existing record with the transformed output map while retaining the original event timestamp.

The Lua filter is registered in Fluent Bit configuration:
```ini
[FILTER]
    Name lua
    Match kube.*
    script /fluent-bit/scripts/clickhouse_transform.lua
    call transform_log
```
##### Lua Script Execution Steps
1. Extracts Kubernetes metadata (`namespace_name`, `pod_name`, `container_name`) from the `kubernetes` map.
2. Extracts raw log string from the `log` field, ensuring safe string casting.
3. Attempts to read structured JSON keys (`level`, `service_name`, `message`, `trace_id`, `span_id`).
4. Normalizes log severity to uppercase string format.
5. Classifies record into `application` (if `service_name` present), `access` (matching HTTP access regex), or `infrastructure`.
6. Formats event timestamp into UTC string with millisecond precision (`YYYY-MM-DD HH:MM:SS.mmm`).
7. Constructs standardized 13-column Lua table matching ClickHouse schema.
8. Returns `2, timestamp, output`, instructing Fluent Bit engine to replace the record while retaining the timestamp.

#### 4.4.4 Output Stage: Direct ClickHouse HTTP Export
Streams transformed records directly to ClickHouse:
```ini
[OUTPUT]
    Name http
    Match kube.*
    Host clickhouse-clickhouse-headless.clickhouse.svc.cluster.local
    Port 8123
    URI /?query=INSERT%20INTO%20observability.kubernetes_logs%20FORMAT%20JSONEachRow
    Format json_lines
```
By utilizing `Format json_lines` combined with ClickHouse's `JSONEachRow` endpoint, Fluent Bit flushes batches of records in a single persistent HTTP connection without intermediate proxy layers.

### 4.5 Deployment Verification & Ingestion Validation
#### 4.5.1 DaemonSet Health & Endpoint Reachability
Fluent Bit was deployed and verified:
```bash
helm upgrade --install fluent-bit \
  ~/clickhouse-unified-poc/helm/fluent-bit-0.58.3.tgz \
  --namespace logging \
  -f ~/clickhouse-unified-poc/configs/fluent-bit-values.yaml \
  --wait --timeout 5m
```
Connectivity from the `logging` namespace to ClickHouse was confirmed by probing the HTTP health endpoint:
```bash
kubectl exec -it -n logging daemonset/fluent-bit -- curl -s http://clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:8123/ping
```
*Output returned:* `Ok.` (HTTP 200).

#### 4.5.2 Database Ingestion Verification
ClickHouse SQL queries verified active record ingestion:
```sql
SELECT count() FROM observability.kubernetes_logs;
```
Initial execution returned 619 records, steadily increasing as container workloads generated logs.

Namespace distribution verified broad cluster log collection:
```sql
SELECT
    namespace,
    count() AS total_logs
FROM observability.kubernetes_logs
GROUP BY namespace
ORDER BY total_logs DESC;
```
*Active namespaces confirmed:* `clickhouse-operator-system`, `logging`, `kube-system`, `clickhouse`, and `applications`.

#### 4.5.3 Application & Trace Correlation Validation
Queries targeting the `applications` namespace confirmed that application-level metadata and tracing keys were successfully extracted and persisted:
```sql
SELECT
    timestamp,
    service_name,
    level,
    message,
    trace_id,
    span_id
FROM observability.kubernetes_logs
WHERE namespace = 'applications'
ORDER BY timestamp DESC
LIMIT 10;
```
Both `order-api` and `payment-service` records demonstrated exact trace identifier extraction (`trace_id: 19f538ceb85f2ab83b81df829f2f6dca`), confirming end-to-end log ingestion integrity.

### 4.6 Phase 4 Validation Summary
| Validation Milestone | Verification Method | Observed Result |
| :--- | :--- | :--- |
| **Log Schema Creation** | `CREATE TABLE observability.kubernetes_logs` | MergeTree table verified |
| **DaemonSet Deployment** | Helm install of Fluent Bit in `logging` | 1/1 Running on Kind node |
| **ClickHouse Connectivity** | Curl to `clickhouse:8123/ping` from logging pod | `HTTP 200 Ok.` |
| **Cluster Log Ingestion** | SQL query `SELECT count() FROM kubernetes_logs` | Initial count: 619 records |
| **Namespace Distribution** | Aggregation query grouping by `namespace` | Logs from `kube-system`, `logging`, `clickhouse` |
| **Application Log Ingestion**| Query filtering `namespace = 'applications'` | Logs from `order-api` & `payment-service` |
| **Error Log Ingestion** | Query filtering `level = 'ERROR'` | Captured `PAYMENT_DECLINED` failure logs |
| **Correlation Metadata** | Verification of `trace_id` and `span_id` | Preserved in ClickHouse table |

## 5. Phase 5: Metrics Collection & Ingestion via OpenTelemetry Collector
### 5.1 Architecture: Eliminating the Dedicated Prometheus Server
A core architectural milestone of this POC is demonstrating that ClickHouse can serve as the primary metrics datastore without deploying, managing, or scaling a dedicated Prometheus Server. 

In traditional observability architectures, Prometheus scrapes endpoints, maintains an on-disk time-series database (TSDB), and evaluates alerting rules. In this modernized topology, the **OpenTelemetry Collector Contrib** distribution replaces the scraping and buffering tiers entirely. Its internal **Prometheus Receiver** handles HTTP polling across infrastructure and application targets, processes metrics through memory-limiting and batching pipelines, and exports rows directly into ClickHouse over native TCP port `9000`:

```text
                  Scrape Targets in Kind Cluster
   +---------------------------------------------------------+
   | Node Exporter (Node CPU, Memory, Disk, Network) :9100   |
   | kube-state-metrics (Pod, Deployment, Node States) :8080  |
   | cAdvisor (Container CPU, Memory, FS) via API Proxy :443 |
   | Order API (orders_created_total, latency) :8000         |
   | Payment Service (payments_processed_total, latency) :8001|
   +---------------------------------------------------------+
                               |
                               | HTTP / HTTPS Scrape (15s - 30s)
                               v
               OpenTelemetry Collector (Deployment)
                   Namespace: opentelemetry
   +---------------------------------------------------------+
   | [RECEIVER: prometheus]                                  |
   |   5 Configured Scrape Jobs                              |
   +---------------------------------------------------------+
                               |
   +---------------------------------------------------------+
   | [PROCESSORS: memory_limiter, batch]                     |
   |   Guards heap memory, batches telemetry chunks          |
   +---------------------------------------------------------+
                               |
   +---------------------------------------------------------+
   | [EXPORTER: clickhouse]                                  |
   |   Native TCP (Port 9000), LZ4 compression, async_insert |
   +---------------------------------------------------------+
                               |
                               | Native TCP (Port 9000)
                               v
                     ClickHouse Server Pod
                     Namespace: clickhouse
   +---------------------------------------------------------+
   | observability Database                                  |
   |  ├── otel_metrics_gauge (Point-in-time measurements)    |
   |  ├── otel_metrics_sum (Cumulative monotonic counters)   |
   |  ├── otel_metrics_histogram (Latency bucket data)       |
   |  ├── otel_metrics_summary (Statistical summaries)       |
   |  └── otel_metrics_exponential_histogram                 |
   +---------------------------------------------------------+
```
```text
                    Kubernetes Cluster
                 clickhouse-unified-poc
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
       Node Exporter   kube-state-metrics  cAdvisor
             |              |              |
             +--------------+--------------+
                            |
                      Prometheus Receiver
                            |
                  OpenTelemetry Collector
                            |
                    ClickHouse Exporter (Native TCP Port 9000)
                            |
                       ClickHouse (Database: observability)
                            |
                 +----------+----------+
                 |          |          |
                 v          v          v
            Gauge Table  Sum Table  Histogram Table
                 ^
                 |
        +--------+---------+
        |                  |
        v                  v
     Order API        Payment Service
      /metrics           /metrics
```
#### 5.1.1 Target Scrape Inventory
| Scrape Target | Metric Type | Scrape Interval | Target Metric Categories |
| :--- | :--- | :--- | :--- |
| **Node Exporter** | Host Infrastructure | 30s | Node CPU, memory, filesystem, network traffic |
| **kube-state-metrics** | K8s State Objects | 30s | Pod phases, deployment replicas, container restarts |
| **cAdvisor** | Container Resources | 30s | Container CPU seconds, memory working set, throttling |
| **Order API** | Application Business | 15s | `orders_created_total`, request duration histograms |
| **Payment Service** | Application Business | 15s | `payments_processed_total`, request duration histograms |

### 5.2 OpenTelemetry Collector RBAC & Kubernetes Proxy Scraping
Accessing container-level cAdvisor metrics in a Kind cluster requires querying the Kubernetes API Server's node proxy endpoint (`/api/v1/nodes/<node-name>/proxy/metrics/cadvisor`). Because the API Server requires authentication and authorization, the Collector's ServiceAccount was granted cluster-level permissions via `configs/otel-collector-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-cadvisor
rules:
  - apiGroups: [""]
    resources: [nodes]
    verbs: [get, list, watch]
  - apiGroups: [""]
    resources: [nodes/proxy]
    verbs: [get]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-cadvisor
subjects:
  - kind: ServiceAccount
    name: otel-collector
    namespace: opentelemetry
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-cadvisor
```
Applying this manifest allowed the Collector to mount its service account token (`/var/run/secrets/kubernetes.io/serviceaccount/token`) and authenticate HTTPS proxy requests directly to `kubernetes.default.svc.cluster.local:443`. Verification was confirmed via:
```bash
kubectl auth can-i get nodes/proxy --as=system:serviceaccount:opentelemetry:otel-collector
```
*Output returned:* `yes`.

### 5.3 Collector Helm Configuration & Pipeline Specification
The OpenTelemetry Collector was deployed using Helm chart `open-telemetry/opentelemetry-collector` version `0.178.3` (runtime image `otel/opentelemetry-collector-contrib:0.163.2`). The complete configuration in `configs/otel-collector-values.yaml` specifies:

```yaml
mode: deployment

image:
  repository: otel/opentelemetry-collector-contrib
  tag: "0.163.2"
  pullPolicy: IfNotPresent

command:
  name: otelcol-contrib

replicaCount: 1

serviceAccount:
  create: true
  name: otel-collector
  automountServiceAccountToken: true

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

config:
  exporters:
    debug: {}
    clickhouse:
      endpoint: tcp://clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000?dial_timeout=10s&compress=lz4
      database: observability
      username: default
      create_schema: true
      async_insert: true
      timeout: 5s

  receivers:
    prometheus:
      config:
        scrape_configs:
          - job_name: node-exporter
            scrape_interval: 30s
            static_configs:
              - targets: ["node-exporter-prometheus-node-exporter.metrics.svc.cluster.local:9100"]
          - job_name: kube-state-metrics
            scrape_interval: 30s
            static_configs:
              - targets: ["kube-state-metrics.metrics.svc.cluster.local:8080"]
          - job_name: order-api
            scrape_interval: 15s
            metrics_path: /metrics
            static_configs:
              - targets: ["order-api.applications.svc.cluster.local:8000"]
          - job_name: payment-service
            scrape_interval: 15s
            metrics_path: /metrics
            static_configs:
              - targets: ["payment-service.applications.svc.cluster.local:8001"]
          - job_name: cadvisor
            scrape_interval: 30s
            scheme: https
            metrics_path: /api/v1/nodes/clickhouse-unified-poc-control-plane/proxy/metrics/cadvisor
            bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
            tls_config:
              ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
              server_name: kubernetes.default.svc
            static_configs:
              - targets: ["kubernetes.default.svc.cluster.local:443"]

  service:
    pipelines:
      metrics:
        receivers:
          - otlp
          - prometheus
        processors:
          - memory_limiter
          - batch
        exporters:
          - clickhouse

      logs:
        receivers:
          - otlp
        processors:
          - memory_limiter
          - batch
        exporters:
          - debug

      traces:
        receivers:
          - otlp
          - jaeger
          - zipkin
        processors:
          - memory_limiter
          - batch
        exporters:
          - debug
```
#### Key Exporter Mechanics
* `endpoint: tcp://...:9000?dial_timeout=10s&compress=lz4`: Transports data using ClickHouse's high-speed native binary protocol with LZ4 stream block compression.
* `create_schema: true`: Directs the exporter to automatically generate the appropriate MergeTree schema definitions in ClickHouse upon receiving new telemetry types.
* `async_insert: true`: Directs ClickHouse to buffer small batch inserts in server memory and flush them to disk periodically, mitigating small-part merge amplification.

### 5.4 Automatic Schema Generation & Metrics Table Archetypes
Upon initial export, the ClickHouse Exporter generated five dedicated metrics tables inside the `observability` database:
| Generated Metric Table | Represented Metric Archetype | Real-World Observability Examples |
| :--- | :--- | :--- |
| `otel_metrics_gauge` | Instantaneous point-in-time scalar values that can arbitrarily increase or decrease. | CPU utilization (`node_cpu_seconds_total`), memory usage, container memory RSS. |
| `otel_metrics_sum` | Cumulative or delta monotonic values tracking accumulated counters over time. | Network bytes transmitted, total HTTP requests, `orders_created_total`. |
| `otel_metrics_histogram` | Sample distribution partitioned into predefined discrete threshold buckets. | Request execution duration (`order_request_duration_seconds`), payload sizes. |
| `otel_metrics_summary` | Pre-calculated statistical quantiles and moving summaries. | Legacy statistical client measurements. |
| `otel_metrics_exponential_histogram` | High-resolution histograms using dynamic exponential bucket spacing. | Advanced latency profiling distributions. |

Inspecting table metadata via `system.tables` confirmed active population:
* `otel_metrics_gauge`: ~46,447 rows
* `otel_metrics_sum`: ~28,697 rows
* `otel_metrics_histogram`: ~40 rows
* `otel_metrics_summary`: ~10 rows
* `otel_metrics_exponential_histogram`: 0 rows

#### 5.4.1 Automatically Generated Metric Tables
| Table Name | MergeTree Engine | Telemetry Data Representation |
| :--- | :--- | :--- |
| `otel_metrics_gauge` | MergeTree | Instantaneous values (CPU usage, memory bytes) |
| `otel_metrics_sum` | MergeTree | Cumulative monotonic counters (order totals, request counts) |
| `otel_metrics_histogram` | MergeTree | Explicit bucket boundaries and observation distributions |
| `otel_metrics_summary` | MergeTree | Statistical quantiles calculated by instrumentation |
| `otel_metrics_exponential_histogram` | MergeTree | Scale-factor exponential bucket distributions |

### 5.5 Quantitative Metrics Verification Across Ingestion Targets
#### 5.5.1 Point-in-Time Gauge Distribution
Inspecting `otel_metrics_gauge` confirmed active metrics ingestion across all five scrape targets:

```sql
SELECT
    ServiceName,
    count() AS total_records,
    uniqExact(MetricName) AS unique_metrics
FROM observability.otel_metrics_gauge
GROUP BY ServiceName
ORDER BY ServiceName;
```
*Observed POC Results:*
| Scrape Service Target | Total Gauge Records | Distinct Unique Metrics |
| :--- | :--- | :--- |
| `cadvisor` | 26,698 | 50 |
| `kube-state-metrics` | 21,892 | 153 |
| `node-exporter` | 11,329 | 270 |
| `order-api` | 364 | 13 |
| `payment-service` | 392 | 13 |

#### 5.5.2 Cumulative Sum Distribution
Inspecting `otel_metrics_sum` validated monotonic counter ingestion:

```sql
SELECT
    ServiceName,
    count() AS total_records,
    uniqExact(MetricName) AS unique_metrics
FROM observability.otel_metrics_sum
GROUP BY ServiceName
ORDER BY ServiceName;
```
*Observed POC Results:*
| Scrape Service Target | Total Sum Records | Distinct Unique Metrics |
| :--- | :--- | :--- |
| `cadvisor` | 33,232 | 44 |
| `kube-state-metrics` | 345 | 1 |
| `node-exporter` | 8,256 | 69 |
| `order-api` | 341 | 5 |
| `payment-service` | 372 | 5 |

#### 5.5.3 Custom Business Counter Analysis
To verify that business metrics emitted by application code reached ClickHouse, custom counters were queried:
```sql
SELECT
    ServiceName,
    MetricName,
    max(TimeUnix) AS latest_timestamp,
    argMax(Value, TimeUnix) AS latest_value
FROM observability.otel_metrics_sum
WHERE MetricName IN ('orders_created_total', 'payments_processed_total')
GROUP BY ServiceName, MetricName
ORDER BY ServiceName;
```
*Output confirmed:* `order-api` recorded `orders_created_total` with latest value `2` (34 stored samples), while `payment-service` recorded `payments_processed_total` with latest value `1` (68 stored samples).

*Critical Analytical Note:* Because Prometheus counters represent running cumulative totals, executing `sum(Value)` across historical samples produces a meaningless mathematical sum of intermediate snapshots (e.g., `68`). True operational analysis requires computing rates of change (`delta()`) or selecting the latest point using `argMax(Value, TimeUnix)`.

#### 5.5.4 Continuous Ingestion Proof (5-Minute Dynamic Window)
To confirm that metrics were actively streaming rather than static relics, a 5-minute sliding window query was executed:
```sql
SELECT
    ServiceName,
    count() AS recent_records,
    uniqExact(MetricName) AS unique_metrics,
    max(TimeUnix) AS latest_timestamp
FROM observability.otel_metrics_gauge
WHERE TimeUnix >= now() - INTERVAL 5 MINUTE
GROUP BY ServiceName
ORDER BY ServiceName;
```
*Result:* Produced **44,370 recent records** across all five services within the 5-minute interval, confirming uninterrupted time-series ingestion.

### 5.6 Phase 5 Validation Summary
| Validation Milestone | Verification Method | Observed Result |
| :--- | :--- | :--- |
| **Collector Deployment** | Helm install of OTel Collector Contrib in `opentelemetry` | 1/1 Running |
| **API Server RBAC** | `ClusterRole` binding for node proxy access | `kubectl auth can-i` returned `yes` |
| **Prometheus Scraping** | Prometheus Receiver scraping 5 targets | All 5 scrape jobs active |
| **Schema Creation** | Exporter `create_schema: true` in ClickHouse | 5 metric tables auto-created |
| **Gauge Metrics Ingestion** | Query `otel_metrics_gauge` across targets | 60,675+ records verified |
| **Sum Metrics Ingestion** | Query `otel_metrics_sum` across targets | 42,246+ records verified |
| **Custom Counters** | Verification of `orders_created_total` & `payments` | Counters actively scraped |
| **Continuous Ingestion** | Dynamic 5-minute time window query | 44,370 records verified |

## 6. Phase 6: Distributed Tracing Implementation & Cross-Service Span Graphing
### 6.1 Distributed Tracing Architecture & OpenTelemetry Core Concepts
In a distributed microservice topology, a single user transaction traverses multiple autonomous service boundaries. When an error occurs or latency spikes, isolated service logs cannot reveal the root cause without an end-to-end trace context. 

Phase 6 implements distributed tracing across `order-api` and `payment-service` by configuring OpenTelemetry SDK instrumentation, routing trace spans through the OpenTelemetry Collector via OTLP/HTTP (`port 4318`), and persisting relational trace graphs inside ClickHouse:

```text
                  Client Invocations
                           |
                           v
        Order API Microservice (applications)
     +------------------------------------------+
     | Root Server Span: POST /orders           |
     |   Internal Span: order.create            |
     |   Client Span: POST /payments (HTTPX)    |
     +------------------------------------------+
                           |
                           | HTTP Request with W3C TraceContext:
                           |   traceparent: 00-{trace_id}-{span_id}-01
                           v
      Payment Service Microservice (applications)
     +------------------------------------------+
     | Server Span: POST /payments              |
     | (Adopts ParentSpanId = Order Client Span)|
     |   Internal Span: payment.process         |
     +------------------------------------------+
                           |
                           +--------------------+
                           |                    |
                           | OTLP/HTTP (:4318)  | OTLP/HTTP (:4318)
                           v                    v
          OpenTelemetry Collector (opentelemetry)
     +--------------------------------------------------+
     | [RECEIVER: otlp] (HTTP Port 4318)                |
     | [PROCESSORS: memory_limiter, batch]              |
     | [EXPORTER: clickhouse] (Native TCP Port 9000)    |
     +--------------------------------------------------+
                           |
                           v
              ClickHouse Server (clickhouse)
     +--------------------------------------------------+
     | observability.otel_traces                        |
     |   TraceId, SpanId, ParentSpanId, Duration,       |
     |   ServiceName, SpanName, SpanKind, StatusCode    |
     +--------------------------------------------------+
```
#### Core Distributed Tracing Primitives
1. **Trace ID (32-character Hexadecimal string):** Represents the entire distributed transaction lifecycle from client request to final response (e.g., `c806177ea23ba29ad09eac931261c486`). Propagated across all participating network calls.
2. **Span ID (16-character Hexadecimal string):** Represents a single contiguous unit of execution within a specific service boundary (e.g., executing a database query, rendering a template, or awaiting an HTTP client response).
3. **Parent Span ID (16-character Hexadecimal string):** Encodes causal lineage by referencing the Span ID of the caller that initiated the current span. A root span has an empty `ParentSpanId`, while downstream spans point to their immediate parent.

```text
                  Client Request
                        |
                        v
                 +-------------+
                 |  Order API  |
                 |  Port 8000  |
                 +-------------+
                        |
                 OpenTelemetry SDK
                        |
                  HTTP Client Call (W3C Header Injection)
                        |
                        v
               +------------------+
               | Payment Service  |
               |    Port 8001     |
               +------------------+
                        |
                 OpenTelemetry SDK
                        |
                  OTLP HTTP Export
                        |
                        v
             +-----------------------+
             |  OTel Collector       |
             |  OTLP Receiver (4318) |
             |  Batch Processor      |
             +-----------------------+
                        |
                 ClickHouse Exporter (Native TCP Port 9000)
                        |
                        v
             +-----------------------+
             |      ClickHouse       |
             |  observability        |
             |  .otel_traces         |
             +-----------------------+
```
### 6.2 Application-Level SDK Instrumentation & Code Patterns
Both applications implement standard OpenTelemetry Python SDK configurations without relying on vendor-locked agents:
* **Order API Instrumentation (`application/order-api/main.py`):**
  ```python
  provider = TracerProvider(
      resource=Resource.create({"service.name": "order-api"})
  )
  OTLP_ENDPOINT = os.getenv("OTEL_EXPORTER_OTLP_TRACES_ENDPOINT")
  if OTLP_ENDPOINT:
      exporter = OTLPSpanExporter(endpoint=OTLP_ENDPOINT)
      provider.add_span_processor(BatchSpanProcessor(exporter))
  trace.set_tracer_provider(provider)

  FastAPIInstrumentor.instrument_app(app)
  HTTPXClientInstrumentor().instrument()
  ```
  `FastAPIInstrumentor` automatically wraps incoming REST endpoints, while `HTTPXClientInstrumentor` intercepts outbound network calls to `payment-service` and injects W3C `traceparent` headers into HTTP request headers.

* **Payment Service Instrumentation (`application/payment-service/main.py`):**
  Uses an identical `TracerProvider` with `service.name: payment-service`. When requests arrive from Order API, `FastAPIInstrumentor` automatically extracts the caller's trace context from HTTP headers, linking internal execution directly into the parent trace.

### 6.3 Collector Traces Pipeline Activation
In Phase 5, the Collector's traces pipeline routed data to a `debug` exporter. In Phase 6, `configs/otel-collector-values.yaml` was updated to route spans into the native ClickHouse exporter:

```yaml
traces:
  receivers:
    - otlp
    - jaeger
    - zipkin
  processors:
    - memory_limiter
    - batch
  exporters:
    - clickhouse
```
The Collector deployment was upgraded via Helm to revision 2:
```bash
helm upgrade otel-collector open-telemetry/opentelemetry-collector \
  --namespace opentelemetry \
  --version 0.178.3 \
  -f ~/clickhouse-unified-poc/configs/otel-collector-values.yaml \
  --wait --timeout 5m
```
Startup verification confirmed that the Collector's OTLP HTTP receiver listening on port `4318` was active and healthy.

### 6.4 Application Deployment Updates: OTLP Endpoint Binding
To direct telemetry export to the Collector, the declarative Kubernetes manifests (`kubernetes/order-api.yaml` and `kubernetes/payment-service.yaml`) were updated to inject the internal cluster endpoint:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
    value: "http://otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318/v1/traces"
```
The updated manifests were verified via client dry-run and applied to the cluster:
```bash
kubectl apply -f kubernetes/order-api.yaml -f kubernetes/payment-service.yaml
kubectl rollout status deployment/order-api -n applications
kubectl rollout status deployment/payment-service -n applications
```
Both pods were restarted, and container environment inspection confirmed that `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` was actively populated.

### 6.5 ClickHouse Trace Schema Architecture
Exporting traces triggered ClickHouse Exporter's `create_schema: true` feature, generating three trace tables in the `observability` database:
* `otel_traces`: The primary MergeTree table storing all span records.
* `otel_traces_trace_id_ts`: An index table mapping `TraceId` to timestamps for fast range lookups.
* `otel_traces_trace_id_ts_mv`: A Materialized View keeping the lookup index synchronized with span inserts.

#### Primary Span Table DDL Specification (`observability.otel_traces`)
Inspecting `DESCRIBE TABLE observability.otel_traces` reveals a rich schema capturing the full OpenTelemetry data model:
| Column Name | ClickHouse Data Type | Functional Description |
| :--- | :--- | :--- |
| `Timestamp` | `DateTime64(9)` | Nanosecond-resolution span start timestamp |
| `TraceId` | `String` | 32-character Hexadecimal distributed trace ID |
| `SpanId` | `String` | 16-character Hexadecimal unique span ID |
| `ParentSpanId` | `String` | Causal reference to parent span ID (empty for root) |
| `TraceState` | `String` | W3C vendor-specific state string |
| `SpanName` | `LowCardinality(String)` | Semantic operation name (e.g., `POST /orders`, `order.create`) |
| `SpanKind` | `LowCardinality(String)` | Span role: `SPAN_KIND_SERVER`, `SPAN_KIND_CLIENT`, `SPAN_KIND_INTERNAL` |
| `ServiceName` | `LowCardinality(String)` | Resource service identifier (`order-api`, `payment-service`) |
| `ResourceAttributes` | `Map(LowCardinality(String), String)` | Host, container, and process environment attributes |
| `ScopeName` | `String` | Instrumentation library name (e.g., `opentelemetry.instrumentation.fastapi`) |
| `ScopeVersion` | `String` | Instrumentation library version |
| `SpanAttributes` | `Map(LowCardinality(String), String)` | Request attributes (`http.status_code`, `http.url`, `error.type`) |
| `Duration` | `UInt64` | Span execution duration in nanoseconds |
| `StatusCode` | `LowCardinality(String)` | OpenTelemetry span status: `STATUS_CODE_UNSET`, `STATUS_CODE_OK`, `STATUS_CODE_ERROR` |
| `StatusMessage` | `String` | Error description narrative when status is error |
| `Events.*` | `Nested(...)` | In-span event timestamps, names, and attributes (stack traces) |
| `Links.*` | `Nested(...)` | Cross-trace relational pointers |

#### 6.5.1 Schema Column Breakdown (`observability.otel_traces`)
| Column Name | Data Type | Operational Role |
| :--- | :--- | :--- |
| `Timestamp` | `DateTime64(9)` | Span start timestamp with nanosecond precision |
| `TraceId` | `String` | Global 32-character distributed trace identifier |
| `SpanId` | `String` | 16-character local span operation identifier |
| `ParentSpanId` | `String` | Identifies caller span initiating operation |
| `TraceState` | `String` | W3C vendor-specific routing state |
| `SpanName` | `LowCardinality(String)` | Semantic operation name (e.g., `POST /orders`) |
| `SpanKind` | `LowCardinality(String)` | `Server`, `Client`, `Internal` |
| `ServiceName` | `LowCardinality(String)` | Application name emitting span |
| `ResourceAttributes`| `Map(String, String)` | Host, container, and process environment tags |
| `ScopeName` | `String` | Instrumentation library name |
| `ScopeVersion` | `String` | Instrumentation library version |
| `SpanAttributes` | `Map(String, String)` | HTTP method, route, status code, error details |
| `Duration` | `UInt64` | Span execution duration in nanoseconds |
| `StatusCode` | `LowCardinality(String)` | `Ok`, `Error`, `Unset` |
| `StatusMessage` | `String` | Error description if execution failed |

### 6.6 Trace Data Validation & Cross-Service Correlation Analysis
#### 6.6.1 Service Span Volume
Traffic was generated via Swagger UI across both successful (`amount: 500`) and failed (`amount: 15000`) checkout flows. ClickHouse SQL queries validated span ingestion across both microservices:

```sql
SELECT
    ServiceName,
    count() AS total_spans,
    uniq(TraceId) AS unique_traces,
    max(Timestamp) AS latest_trace
FROM observability.otel_traces
GROUP BY ServiceName
ORDER BY ServiceName;
```
*Observed Validation Snapshot:* `order-api` produced 90 total spans across 28 unique traces, while `payment-service` produced 79 total spans across 25 unique traces.

#### 6.6.2 Identifying Multi-Service Transactions
To prove that traces spanned across service boundaries, queries isolated transactions containing spans from both microservices:
```sql
SELECT
    TraceId,
    groupUniqArray(ServiceName) AS services,
    count() AS total_spans,
    min(Timestamp) AS first_span,
    max(Timestamp) AS last_span
FROM observability.otel_traces
WHERE Timestamp >= now64(9) - INTERVAL 15 MINUTE
GROUP BY TraceId
HAVING uniqExact(ServiceName) = 2
ORDER BY last_span DESC
LIMIT 10;
```
The query isolated shared distributed trace IDs, notably `c806177ea23ba29ad09eac931261c486` and `be75bf4ba315b1dfac9094275ebaafb8`, each containing 11 spans distributed across `['order-api', 'payment-service']`.

#### 6.6.3 Deep Causal Span Tree Reconstruction
Examining Trace ID `c806177ea23ba29ad09eac931261c486` demonstrates exact parent-child span reconstruction and error propagation across the network boundary:

```sql
SELECT
    ServiceName,
    SpanName,
    SpanKind,
    SpanId,
    ParentSpanId,
    Duration,
    StatusCode
FROM observability.otel_traces
WHERE TraceId = 'c806177ea23ba29ad09eac931261c486'
ORDER BY Timestamp;
```
*Observed Execution Trace Hierarchy:*
| Service | Semantic Span Name | Span Kind | Span ID | Parent Span ID | Status | Duration |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `order-api` | `POST /orders` | Server | `4f91d27941f47f88` | *(Root - Empty)* | `Error` | 36.74 ms (`36741827 ns`) |
| `order-api` | `order.create` | Internal | `5617ed65019f6f22` | `4f91d27941f47f88` | `Error` | 34.12 ms |
| `order-api` | `POST` | Client | `aa90914507340553` | `5617ed65019f6f22` | `Error` | 18.25 ms |
| `payment-service` | `POST /payments` | Server | `a26a453bbcf89e54` | **`aa90914507340553`** | `Unset` | 14.50 ms |
| `payment-service` | `payment.process` | Internal | `1d3afae2e0e54ef1` | `a26a453bbcf89e54` | `Error` | 6.80 ms |

#### Critical Architectural Insights
1. **Parent-Child Linkage Across Nodes:** The Payment Service root span (`POST /payments`, Span ID `a26a453bbcf89e54`) explicitly records its `ParentSpanId` as **`aa90914507340553`**, which corresponds exactly to the Order API HTTP client span. This confirms that W3C trace context was successfully injected by HTTPX in Order API, transmitted over HTTP, and decoded by FastAPI in Payment Service.
2. **Causal Error Isolation:** The internal span `payment.process` failed due to the amount constraint (`>= 10000`), marking its status as `Error`. This failure cascaded upward to the parent client call (`POST`) in Order API, and finally marked the top-level transaction (`POST /orders`) as `Error`, precisely mirroring real-world distributed fault behavior.
3. **Latency Accounting:** Total transaction duration was 36.74 ms, of which 18.25 ms was spent in the outbound payment call and 14.50 ms inside the downstream Payment Service, validating ClickHouse's nanosecond duration tracking for microservice performance profiling.

### 6.7 Phase 6 Validation Summary
| Validation Milestone | Verification Method | Observed Result |
| :--- | :--- | :--- |
| **Collector Trace Pipeline** | Updated pipeline to ClickHouse exporter | Upgraded to Revision 2 |
| **OTLP Endpoint Injection** | Set `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | Injected into both Deployments |
| **Trace Table Creation** | Exporter `create_schema: true` | `otel_traces` auto-created |
| **Order API Trace Ingestion**| Query `ServiceName = 'order-api'` | 90 spans across 28 traces |
| **Payment Service Ingestion**| Query `ServiceName = 'payment-service'` | 79 spans across 25 traces |
| **Distributed Trace Linkage**| Query `HAVING uniqExact(ServiceName) = 2` | Identified common Trace IDs |
| **Parent-Child Causality** | Matched ParentSpanId to client SpanId | Proven: `aa90914507340553` |
| **Error Status Propagation** | Query `StatusCode = 'Error'` | Caught `PAYMENT_DECLINED` failure |

## 7. Phase 7: Grafana Visualization, SQL Exploration & Unified Cross-Signal Correlation
### 7.1 Single-Pane-of-Glass Observability Architecture
The final phase of this POC integrates Grafana as the unified querying and visualization interface. Rather than opening separate tools for logs (e.g., Kibana), metrics (e.g., Prometheus UI), and traces (e.g., Jaeger UI), engineers query all three observability signals from a single ClickHouse datasource using standard SQL:

```text
                  Grafana Dashboard & Explore Interface
                                    |
                        ClickHouse-POC Datasource
            (HTTP Port 8123: clickhouse-clickhouse-headless)
                                    |
                                    v
                     ClickHouse Server Pod (26.10.4)
                       observability Database
        +---------------------------+---------------------------+
        |                           |                           |
        v                           v                           v
  kubernetes_logs           otel_metrics_*                 otel_traces
  (Container Logs)         (Gauge/Sum/Hist)            (Distributed Spans)
        ^                           ^                           ^
        |                           |                           |
   Fluent Bit                OTel Collector               OTel Collector
   (DaemonSet)            (Prometheus Receiver)          (OTLP Receiver)
        ^                           ^                           ^
        |                           |                           |
 Kubernetes Pods         Exporters & /metrics         Microservice SDKs
```
```text
                  Kubernetes Cluster
                 clickhouse-unified-poc
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     Fluent Bit       OTel Collector    Application SDKs
     (DaemonSet)     Prometheus Receiver   OTLP HTTP
          |                |                |
          |                +--------+-------+
          |                         |
          v                         v
     ClickHouse HTTP           ClickHouse TCP
       Port 8123                 Port 9000
          |                         |
          +------------+------------+
                       |
                       v
                  ClickHouse
             (observability db)
                       |
             +---------+---------+
             |         |         |
             v         v         v
        Logs Table  Metrics   Traces
             |         |         |
             +---------+---------+
                       |
                       v
                    Grafana
          (ClickHouse-POC Datasource)
```
### 7.2 Grafana Helm Deployment & Security Configuration
#### 7.2.1 Helm Values Specification
Grafana was deployed into the `monitoring` namespace using official Helm chart `grafana/grafana` version `10.5.15` (application version `12.3.1`). Settings were defined in `configs/grafana-values.yaml`:

```yaml
replicas: 1

image:
  repository: grafana/grafana
  tag: "12.3.1"
  pullPolicy: IfNotPresent

adminUser: admin
adminPassword: "admin"

service:
  enabled: true
  type: ClusterIP
  port: 80
  targetPort: 3000

persistence:
  type: pvc
  enabled: true
  storageClassName: standard
  accessModes: ["ReadWriteOnce"]
  size: 5Gi

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```
#### 7.2.2 Installation & Readiness Verification
The Helm chart was rendered to `/tmp/grafana-rendered.yaml` to validate generated resources before deployment:
```bash
helm install grafana grafana/grafana \
  --version 10.5.15 \
  --namespace monitoring \
  -f ~/clickhouse-unified-poc/configs/grafana-values.yaml
```
During container initialization, Grafana briefly showed `0/1 Running` while executing internal SQLite schema migrations. Once logs reported `HTTP Server Listen address=[::]:3000 protocol=http` and `All modules healthy`, the pod transitioned to `1/1 Running`. The `5Gi` PersistentVolumeClaim successfully bound to the Kind cluster's `standard` local storage class.

#### 7.2.3 Credential Recovery & Local Access
Administrator credentials were verified from the Kubernetes Secret:
```bash
kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-user}" | base64 --decode; echo
kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```
Local browser access was established via port forwarding:
```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```
Grafana was accessible at `http://localhost:3000` using credentials `admin` / `admin`.

### 7.3 ClickHouse Datasource Configuration
Grafana connects to ClickHouse via the official ClickHouse datasource plugin using the internal Kubernetes DNS name:
| Datasource Parameter | Configured Value | Architectural Context |
| :--- | :--- | :--- |
| **Data Source Name** | `ClickHouse-POC` | Unique identifier for dashboard queries |
| **Host / Address** | `clickhouse-clickhouse-headless.clickhouse.svc.cluster.local` | Internal headless service DNS in `clickhouse` namespace |
| **Port** | `8123` | ClickHouse HTTP interface port |
| **Protocol** | `HTTP` | Enables HTTP query transmission from Grafana server |
| **Default Database** | `observability` | Target database containing all observability tables |
| **User / Password** | `default` / *(empty)* | Unauthenticated local POC access |

Testing the datasource connection returned a green confirmation, establishing that Grafana could execute SQL queries directly against ClickHouse.

#### 7.3.1 Datasource Configuration Parameters
| Parameter | Configured Value | Operational Purpose |
| :--- | :--- | :--- |
| **Datasource Name** | `ClickHouse-POC` | Identifier used in dashboard queries |
| **Host Address** | `clickhouse-clickhouse-headless.clickhouse.svc.cluster.local` | Internal headless service DNS |
| **Port** | `8123` | ClickHouse HTTP interface port |
| **Protocol** | `HTTP` | Transport protocol used by Grafana ClickHouse plugin |
| **Database** | `observability` | Centralized telemetry database |
| **Username** | `default` | ClickHouse database user |

### 7.4 Multi-Signal Query Validation in Grafana Explore
#### 7.4.1 Exploring Kubernetes Container Logs
Navigating to **Explore** under datasource `ClickHouse-POC`, log ingestion was verified:
```sql
SELECT
    timestamp,
    namespace,
    pod_name,
    container_name,
    service_name,
    level,
    message,
    trace_id
FROM observability.kubernetes_logs
ORDER BY timestamp DESC
LIMIT 20;
```
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/00bde23a-55f1-42b1-97e2-34e70fc12a88" />

*Results:* Returned live log streams across `kube-system`, `logging`, and `applications`. 
*Handling `UNKNOWN` Severity:* In plain-text system containers where logs lack JSON severity tags, the Lua transform assigns `level = UNKNOWN`. The complete unparsed narrative remains accessible via `raw_log`, guaranteeing zero data loss.

Filtering specifically for microservice logs confirmed structured data extraction:
```sql
SELECT
    timestamp,
    service_name,
    level,
    message,
    trace_id
FROM observability.kubernetes_logs
WHERE namespace = 'applications'
ORDER BY timestamp DESC
LIMIT 20;
```
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/8a94f919-904d-4f01-84bf-ba9708e220a6" />

Both `order-api` and `payment-service` records demonstrated exact trace identifier extraction.

#### 7.4.2 Exploring Infrastructure and Application Metrics
Gauges and cumulative counters were queried across scrape jobs:
```sql
SELECT
    ServiceName,
    MetricName,
    Value,
    TimeUnix
FROM observability.otel_metrics_gauge
WHERE MetricName != ''
ORDER BY TimeUnix DESC
LIMIT 20;
```
Samples confirmed continuous data points from `node-exporter`, `cadvisor`, `kube-state-metrics`, `order-api`, and `payment-service`.

Application counter metrics for `order-api` and `payment-service` were analyzed:
```sql
SELECT
    ServiceName,
    MetricName,
    Value,
    TimeUnix
FROM observability.otel_metrics_sum
WHERE ServiceName IN ('order-api', 'payment-service')
ORDER BY TimeUnix DESC
LIMIT 20;
```
*Analytical Rationale:* Because counter tables record consecutive snapshots of cumulative metrics (`orders_created_total`), meaningful dashboard visualizations should apply `nonNegativeDerivative()` or delta calculations between successive timestamps rather than raw values.

#### 7.4.3 Exploring Distributed Traces
Trace records were queried from `observability.otel_traces`:
```sql
SELECT
    Timestamp,
    ServiceName,
    SpanName,
    TraceId,
    SpanId,
    ParentSpanId,
    Duration,
    StatusCode,
    StatusMessage
FROM observability.otel_traces
ORDER BY Timestamp DESC
LIMIT 20;
```
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/dea5da24-639c-413d-98b9-1058baf3e254" />

*Service-Level Trace Totals:* Cumulative telemetry revealed extensive trace accumulation: `order-api` generated 1,443 total spans across 479 unique traces, while `payment-service` produced 1,432 total spans across 476 unique traces.

*Note on Prometheus Scrape Spans:* Spans representing `GET /metrics` were visible in the trace stream. These are automatically generated by FastAPI when the OpenTelemetry Collector Prometheus Receiver scrapes the application endpoints. Production dashboards can filter out `/metrics` spans by adding `WHERE SpanName != 'GET /metrics'`.

### 7.5 Cross-Signal Correlation: Linking Traces to Logs via Trace ID
The crowning capability demonstrated in this POC is **bi-directional correlation across signals** within a single SQL database.

#### Step 1: Identifying Distributed Transactions
Executing a multi-service aggregation isolated distributed traces traversing both microservices:
```sql
SELECT
    TraceId,
    groupUniqArray(ServiceName) AS services,
    count() AS total_spans,
    min(Timestamp) AS first_seen,
    max(Timestamp) AS last_seen
FROM observability.otel_traces
WHERE Timestamp >= now() - INTERVAL 24 HOURS
GROUP BY TraceId
HAVING uniq(ServiceName) >= 2
ORDER BY last_seen DESC
LIMIT 10;
```
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/8f7015f4-ade7-4962-95a2-b5b2b4bc56a3" />

Selecting common Trace ID **`c806177ea23ba29ad09eac931261c486`** provides the correlation key.

#### Step 2: Fetching the Correlated Log Narrative
Using the Trace ID, all related application logs across both services were retrieved in exact chronological sequence:
```sql
SELECT
    timestamp,
    service_name,
    level,
    message,
    trace_id
FROM observability.kubernetes_logs
WHERE trace_id = 'c806177ea23ba29ad09eac931261c486'
ORDER BY timestamp ASC;
```
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/217ebe28-4d17-46db-bc52-f54a7ca8e9b0" />

*Correlated Execution Narrative:*
| Sequence | Service | Severity | Emitted Log Message | Trace Context |
| :---: | :--- | :--- | :--- | :--- |
| **1** | `order-api` | `INFO` | Order created successfully | `trace_id: c806177ea...` |
| **2** | `payment-service` | `INFO` | Payment processing started | `trace_id: c806177ea...` |
| **3** | `payment-service` | `ERROR` | Payment processing failed (`PAYMENT_DECLINED`) | `trace_id: c806177ea...` |
| **4** | `order-api` | `ERROR` | Payment processing failed (`Client error '400'`) | `trace_id: c806177ea...` |

#### Step 3: Fetching the Correlated Trace Span Tree
Querying `observability.otel_traces` with the exact same Trace ID reveals the underlying execution mechanics:
```sql
SELECT
    Timestamp,
    ServiceName,
    SpanName,
    SpanKind,
    SpanId,
    ParentSpanId,
    Duration,
    StatusCode
FROM observability.otel_traces
WHERE TraceId = 'c806177ea23ba29ad09eac931261c486'
ORDER BY Timestamp ASC;
```
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/e4c1431b-3b49-4d6c-8d91-44d99af38e21" />

*Reconstructed Execution Tree:*
```text
Trace ID: c806177ea23ba29ad09eac931261c486
Duration: 36.74 ms | Status: Error

Order API
  └── [Server] POST /orders (SpanId: 4f91d27941f47f88) - Status: Error
        └── [Internal] order.create (SpanId: 5617ed65019f6f22) - Status: Error
              └── [Client] POST /payments (SpanId: aa90914507340553) - Status: Error
                    │
                    └── (Network HTTP Call)
                          │
Payment Service           v
  └── [Server] POST /payments (SpanId: a26a453bbcf89e54, ParentSpanId: aa90914507340553)
        └── [Internal] payment.process (SpanId: 1d3afae2e0e54ef1) - Status: Error
```
#### Diagnostic Synthesis
The causal timeline is completely clear:
1. At $t_0$, `order-api` accepts the checkout request and starts `POST /orders`.
2. At $t_1$, `order-api` calls `payment-service` with `amount: 15000`.
3. At $t_2$, `payment-service` evaluates the business rule (`amount >= 10000`), logs an `ERROR` with `error_type: "PAYMENT_DECLINED"`, and marks span `payment.process` as `Error`.
4. At $t_3$, `payment-service` returns HTTP 400.
5. At $t_4$, `order-api` catches the HTTP 400 client exception, marks its outbound client span and parent server span as `Error`, logs an `ERROR`, and returns HTTP 502 to the user.

An on-call engineer can jump from an error spike in a metric dashboard directly to the trace graph, and from that trace graph pull all corresponding application logs within seconds using a single database.

### 7.6 Phase 7 Comprehensive Validation Matrix
| Architectural Validation Dimension | Executed Verification Mechanism | Observed Validation Outcome | Status |
| :--- | :--- | :--- | :--- |
| **Grafana Deployment** | Helm install in `monitoring`, pod health inspection | Pod `1/1 Running`, `All modules healthy` | Successful |
| **Persistence Layer** | PVC inspection in `monitoring` | `5Gi` standard StorageClass bound | Successful |
| **Credential Retrieval** | Base64 decode of `grafana` Secret | Admin username and password retrieved cleanly | Successful |
| **Port Forwarding Access** | Port forward `3000:80`, browser login | Authenticated access to Grafana UI | Successful |
| **Datasource Integration** | ClickHouse HTTP plugin connection on port `8123` | `ClickHouse-POC` data source tested and healthy | Successful |
| **Kubernetes Log Ingestion** | Explore SQL query on `observability.kubernetes_logs` | Queried container logs across all cluster namespaces | Successful |
| **Application Log Ingestion** | Explore query filtering `namespace = 'applications'` | Extracted structured fields and trace correlation IDs | Successful |
| **Continuous Metrics** | Explore query on `otel_metrics_gauge` & `_sum` | Verified active time-series streams from 5 targets | Successful |
| **Distributed Trace Ingestion**| Explore query on `observability.otel_traces` | Queried spans across `order-api` and `payment-service` | Successful |
| **Cross-Service Trace Linkage**| `HAVING uniq(ServiceName) >= 2` grouping | Identified distributed traces with spans in both services | Successful |
| **Parent-Child Causality** | ParentSpanId verification across services | Matched `order-api` client span to `payment-service` server span | Successful |
| **Bi-directional Correlation** | Shared Trace ID query matching logs and traces | Correlated 4 sequential logs with 11-span trace tree | Successful |

### 7.7 Technical Limitations & Production Hardening Roadmap
While this POC successfully proves the technical feasibility of unified observability in ClickHouse, migrating this architecture to enterprise production requires addressing specific operational boundaries:

1. **High Availability & Clustering:** The POC operates on a single Kind node with single replicas for ClickHouse, Keeper, OTel Collector, and Grafana. Enterprise production requires a 3-node ClickHouse Keeper quorum, multi-shard/multi-replica topologies utilizing `ReplicatedMergeTree` engines, horizontal collector autoscaling, and multi-replica Grafana backed by external PostgreSQL.
2. **Security & Transport Encryption:** Currently operates with plain HTTP transport and unauthenticated `default` ClickHouse access. Production hardening requires enforcing mTLS via `cert-manager` across native (`9440`) and HTTPS (`8443`) ports, configuring role-based database user privileges, and integrating HashiCorp Vault or Kubernetes External Secrets.
3. **Data Lifecycle, Retention & Downsampling:** Ingestion currently grows append-only bounded by PVC capacity. Production deployments must establish declarative table `TTL` rules (e.g., 30d logs, 14d traces, 7d raw metrics) and Materialized Views to downsample high-frequency metrics into 5-minute and 1-hour rollups.
4. **Log Pipeline Enhancements:** Plain-text system logs currently receive `level = UNKNOWN`, and unmapped dynamic fields remain in `{}`. Production requires enhanced Lua regex parsers for heterogeneous access formats and unpacking application JSON fields (`order_id`, `amount`, `customer_id`) into typed columns or indexed `Map(String, String)` columns.
5. **Tracing Scale & Tail-Based Sampling:** Evaluated using 100% trace sampling, which captures non-critical Prometheus `/metrics` scrapes. Production requires deploying OpenTelemetry Collector `tail_sampling` processors to capture 100% of errors and high-latency paths while sampling nominal transactions at 1–5%, alongside filtering internal health/scrape endpoints.
6. **Ingestion Buffering & Backpressure:** Direct collector-to-database writes expose ClickHouse to ingestion spikes. Production architectures should introduce an Apache Kafka or Redpanda message queue as a persistent buffer, leveraging ClickHouse's native Kafka Engine to decouple ingestion spikes from disk part merges.

### 7.8 Final Conclusion
This Proof of Concept successfully demonstrates that **ClickHouse can serve as a unified, high-performance analytical storage engine for logs, metrics, and traces**. By orchestrating Fluent Bit for direct log forwarding, the OpenTelemetry Collector for Prometheus scraping and OTLP trace export, and Grafana for SQL-driven visualization, the architecture eliminates multi-backend operational silos. Most importantly, shared trace identifiers bridge the gap between application narratives and distributed call trees, providing deep, correlated system visibility within a single unified platform.
