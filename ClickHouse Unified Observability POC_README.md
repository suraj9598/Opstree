# ClickHouse Unified Observability POC — Complete Setup Guide

## 1. Purpose of This Document

This README is a complete setup and reproduction guide for the ClickHouse Unified Observability POC performed on a local Kubernetes cluster using Kind.

In this POC, two environments are kept separate:

1. **ClickHouse-based observability stack**
2. **Traditional observability stack**

---

# Part 1 — ClickHouse Unified Observability Stack

## 2. Architecture

The ClickHouse POC uses ClickHouse as the common storage backend for logs, metrics, and traces.

```text
                         Kubernetes / Kind
                                |
          +---------------------+----------------------+
          |                     |                      |
        Logs                  Metrics                Traces
          |                     |                      |
     Fluent Bit        Node Exporter               Test App
          |             Kube-State-Metrics             |
          |             Kubelet/cAdvisor               |
          |                     |                      |
          |              OpenTelemetry Collector <-----+
          |                     |
          +-----------> ClickHouse <-------------------+
                         |
                  observability DB
                         |
          +--------------+---------------+
          |              |               |
      Logs table     Metrics tables   Traces table
          |              |               |
          +--------------+---------------+
                         |
                       Grafana
```

### Components

| Component | Purpose |
|---|---|
| Kind | Creates the local Kubernetes cluster |
| cert-manager | Provides Kubernetes certificate management/webhook support |
| ClickHouse Operator | Manages ClickHouse resources in Kubernetes |
| ClickHouse Keeper | Provides coordination for ClickHouse |
| ClickHouse | Stores logs, metrics, and traces |
| Fluent Bit | Collects Kubernetes container logs |
| Node Exporter | Exposes node-level operating system metrics |
| Kube-State-Metrics | Exposes Kubernetes object/state metrics |
| Kubelet/cAdvisor | Exposes container resource metrics |
| OpenTelemetry Collector | Scrapes metrics and receives OTLP traces |
| Grafana | Visualizes ClickHouse data |
| SSH reverse tunnel | Makes the local Kind services reachable from Grafana on AWS |

---

# 3. Prerequisites

The POC was performed on a local Linux laptop.

The environment used approximately:

```text
CPU:       8 logical CPUs
RAM:       15 GiB
Disk:      ~468 GiB
Docker:    29.7.2
kubectl:   1.29.15 client
Kind:      0.33.0
Helm:      4.3.0
```

Install/check the required tools:

```bash
docker --version
kubectl version --client
kind version
helm version
```

Verify Docker is running:

```bash
docker ps
```

> **Important:** An existing company Kubernetes context was present on the laptop. The POC was created in a separate Kind cluster. Do not modify the company `readonly-context`.

---

# 4. Create the Kind Kubernetes Cluster

Create the Kind configuration:

```bash
cat > kind-clickhouse.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kind-clickhouse-poc
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF
```

Create the cluster:

```bash
kind create cluster --config kind-clickhouse.yaml
```

Verify:

```bash
kubectl get nodes
```

Expected result:

```text
kind-clickhouse-poc-control-plane   Ready
kind-clickhouse-poc-worker          Ready
kind-clickhouse-poc-worker2         Ready
```

Check the current context:

```bash
kubectl config current-context
```

Expected:

```text
kind-kind-clickhouse-poc
```

---

# 5. Kubernetes Storage

Kind provides a default StorageClass using local-path provisioning.

Check it:

```bash
kubectl get storageclass
```

Expected:

```text
NAME                 PROVISIONER
standard (default)   rancher.io/local-path
```

This storage is suitable for the local POC. It is not equivalent to production-grade replicated storage.

---

# 6. Create the ClickHouse Namespace

```bash
kubectl create namespace clickhouse
```

Verify:

```bash
kubectl get namespace clickhouse
```

---

# 7. Install cert-manager

cert-manager was installed because the Kubernetes environment used certificate/webhook-related functionality.

Install:

```bash
helm install cert-manager \
  --create-namespace \
  --namespace cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --set crds.enabled=true \
  --version v1.19.2
```

Verify:

```bash
kubectl get pods -n cert-manager
```

All cert-manager pods should become `Running`.

---

# 8. Install the ClickHouse Operator

The ClickHouse Operator manages ClickHouse custom resources and creates the required Kubernetes objects.

Install:

```bash
helm install clickhouse-operator \
  --create-namespace \
  -n clickhouse-operator-system \
  oci://ghcr.io/clickhouse/clickhouse-operator-helm
```

Verify:

```bash
kubectl get pods -n clickhouse-operator-system
```

Check CRDs:

```bash
kubectl get crd | grep clickhouse
```

Expected CRDs include:

```text
clickhouseclusters.clickhouse.com
keeperclusters.clickhouse.com
```

---

# 9. ClickHouse Operator DNS Issue

During the initial deployment, the operator reported errors similar to:

```text
failed to probe replica
dial tcp: lookup clickhouse-clickhouse-internal-0-0.clickhouse.svc.cluster.local: i/o timeout
```

The issue was related to DNS lookup behavior in the isolated Kind lab.

A lab-specific DNS configuration was used in CoreDNS:

```text
forward . 8.8.8.8 1.1.1.1 {
  max_concurrent 1000
}
```

> **Lab note:** Do not blindly copy public DNS forwarding into production Kubernetes. Production DNS should normally follow the organization's DNS architecture.

The operator deployment was also configured with:

```yaml
dnsConfig:
  options:
    - name: ndots
      value: "1"
```

Patch:

```bash
kubectl patch deployment clickhouse-operator-controller-manager \
  -n clickhouse-operator-system \
  --type='strategic' \
  -p='{"spec":{"template":{"spec":{"dnsConfig":{"options":[{"name":"ndots","value":"1"}]}}}}}'
```

Verify rollout:

```bash
kubectl rollout status deployment/clickhouse-operator-controller-manager \
  -n clickhouse-operator-system
```

---

# 10. Deploy ClickHouse Keeper

ClickHouse Keeper provides coordination functionality used by ClickHouse.

For the low-resource POC, one Keeper replica was used.

Create `keeper.yaml`:

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: KeeperCluster
metadata:
  name: clickhouse-keeper
  namespace: clickhouse
spec:
  replicas: 1
  dataVolumeClaimSpec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 2Gi
```

Apply:

```bash
kubectl apply -f keeper.yaml
```

Check:

```bash
kubectl get keepercluster -n clickhouse
```

Expected status:

```text
READY   True
STATUS  Standalone Keeper is ready
```

Check pod:

```bash
kubectl get pods -n clickhouse
```

Expected:

```text
clickhouse-keeper-keeper-0-0   Running
```

Check PVC:

```bash
kubectl get pvc -n clickhouse
```

---

# 11. Deploy ClickHouse

The ClickHouse cluster used one shard and one replica for the resource-constrained POC.

Create `clickhouse.yaml`:

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: ClickHouseCluster
metadata:
  name: clickhouse
  namespace: clickhouse
spec:
  shards: 1
  replicas: 1

  keeperClusterRef:
    name: clickhouse-keeper

  containerTemplate:
    image:
      repository: docker.io/clickhouse/clickhouse-server
      tag: latest
    imagePullPolicy: IfNotPresent

    resources:
      requests:
        cpu: 250m
        memory: 1Gi
      limits:
        cpu: "1"
        memory: 2Gi

  dataVolumeClaimSpec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
```

Apply:

```bash
kubectl apply -f clickhouse.yaml
```

Check:

```bash
kubectl get clickhousecluster -n clickhouse
```

Check pods:

```bash
kubectl get pods -n clickhouse
```

Check services:

```bash
kubectl get svc -n clickhouse
```

---

# 12. ClickHouse Memory Issue

An earlier ClickHouse deployment used a memory limit of only 512Mi.

The ClickHouse server automatically calculated a lower effective memory limit, approximately:

```text
460.8 MiB
```

This caused insert failures such as:

```text
Code: 241
Memory limit exceeded
```

It also caused probe timeouts and restarts when the OpenTelemetry Collector inserted data.

The host itself was not out of memory.

The solution was to increase the ClickHouse container memory limit:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 1Gi
  limits:
    cpu: "1"
    memory: 2Gi
```

After applying the corrected configuration, ClickHouse became stable.

Verify:

```bash
kubectl get pods -n clickhouse
```

---

# 13. Connect to ClickHouse

The ClickHouse client was available in the ClickHouse pod.

Find the pod:

```bash
kubectl get pods -n clickhouse
```

Execute the client:

```bash
kubectl exec -it \
  -n clickhouse \
  <clickhouse-pod-name> \
  -- clickhouse-client
```

Verify:

```sql
SELECT version();
```

Check databases:

```sql
SHOW DATABASES;
```

---

# 14. Create the Observability Database

```sql
CREATE DATABASE observability;
```

Verify:

```sql
SHOW DATABASES;
```

---

# 15. Understand the ClickHouse Table Model

ClickHouse stores data in tables.

For the POC, separate tables were used for different signals.

```text
observability
├── kubernetes_logs
├── otel_metrics_gauge
├── otel_metrics_sum
├── otel_metrics_histogram
├── otel_metrics_summary
├── otel_metrics_exponential_histogram
└── otel_traces
```

The metrics and trace tables were created by the OpenTelemetry ClickHouse exporter.

---

# 16. Create a Simple Learning Table

This table was created to understand ClickHouse basics before connecting real observability data.

```sql
CREATE TABLE observability.logs
(
    timestamp DateTime,
    service String,
    level String,
    message String
)
ENGINE = MergeTree
ORDER BY timestamp;
```

Insert sample data:

```sql
INSERT INTO observability.logs VALUES
('2026-09-28 10:00:00', 'employee-api', 'INFO', 'Employee created'),
('2026-09-28 10:01:00', 'attendance-api', 'INFO', 'Attendance recorded'),
('2026-09-28 10:02:00', 'salary-api', 'ERROR', 'Salary calculation failed');
```

Query:

```sql
SELECT *
FROM observability.logs
ORDER BY timestamp;
```

Filter:

```sql
SELECT *
FROM observability.logs
WHERE level = 'ERROR';
```

Group:

```sql
SELECT service, count()
FROM observability.logs
GROUP BY service;
```

---

# 17. Understand MergeTree

`MergeTree` is one of the main ClickHouse table engines.

Example:

```sql
ENGINE = MergeTree
ORDER BY timestamp
```

The `ORDER BY` expression determines the primary sorting key used by ClickHouse.

ClickHouse stores inserted data in parts and merges those parts in the background.

Inspect parts:

```sql
SELECT
    database,
    table,
    count() AS parts,
    sum(rows) AS rows,
    sum(bytes_on_disk) AS bytes
FROM system.parts
WHERE database = 'observability'
  AND active = 1
GROUP BY database, table;
```

---

# 18. Deploy Fluent Bit for ClickHouse Logs

Fluent Bit collects Kubernetes container logs.

The logs are available on Kubernetes nodes under:

```text
/var/log/containers/
```

The ClickHouse Fluent Bit configuration used:

```yaml
config:
  customParsers: |
    [PARSER]
        Name docker_no_time
        Format json
        Time_Keep Off
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L

  extraFiles: {}

  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [FILTER]
        Name nest
        Match kube.*
        Operation lift
        Nested_under kubernetes

    [FILTER]
        Name modify
        Match kube.*
        Rename namespace_name namespace
        Rename pod_name pod
        Rename container_name container
        Rename host node
        Rename log message

  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Exclude_Path /var/log/containers/fluent-bit-*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
        Inotify_Watcher false

    [INPUT]
        Name systemd
        Tag host.*
        Systemd_Filter _SYSTEMD_UNIT=kubelet.service
        Read_From_Tail On

  outputs: |
    [OUTPUT]
        Name http
        Match kube.*
        Host clickhouse-clickhouse-headless.clickhouse.svc.cluster.local
        Port 8123
        URI /?query=INSERT%20INTO%20observability.kubernetes_logs%20FORMAT%20JSONEachRow
        Format json_lines
        Header Content-Type application/json
```

The output uses ClickHouse HTTP port:

```text
8123
```

The ClickHouse native protocol uses:

```text
9000
```

For Fluent Bit, the HTTP API was used because the Fluent Bit build did not provide a native ClickHouse output plugin.

---

# 19. Create the Kubernetes Logs Table

```sql
CREATE TABLE observability.kubernetes_logs
(
    timestamp DateTime,
    namespace String,
    pod String,
    container String,
    node String,
    message String
)
ENGINE = MergeTree
ORDER BY timestamp;
```

The Fluent Bit HTTP output sends JSONEachRow records directly to this table.

Verify:

```sql
SELECT count()
FROM observability.kubernetes_logs;
```

Check recent logs:

```sql
SELECT
    timestamp,
    namespace,
    pod,
    container,
    node,
    message
FROM observability.kubernetes_logs
ORDER BY timestamp DESC
LIMIT 20;
```

---

# 20. Fluent Bit Inotify / File Descriptor Problem

During the POC, Fluent Bit pods on Kind workers encountered:

```text
Too many open files
```

Example:

```text
tail_fs_inotify.c errno=24 Too many open files
```

The configuration was changed to:

```text
Inotify_Watcher false
```

This allowed Fluent Bit to operate without the problematic inotify watcher behavior.

After configuration, restart/redeploy Fluent Bit and verify:

```bash
kubectl get pods -n logging
```

Expected:

```text
fluent-bit-...   1/1   Running
```

Check logs:

```bash
kubectl logs -n logging <fluent-bit-pod>
```

The ClickHouse HTTP output should return:

```text
status=200
```

---

# 21. Node Exporter

Node Exporter exposes operating-system metrics such as:

- CPU
- Memory
- Disk
- Network

Add the Prometheus Community repository if required:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install:

```bash
helm install node-exporter \
  prometheus-community/prometheus-node-exporter \
  -n monitoring \
  --create-namespace
```

Verify:

```bash
kubectl get pods -n monitoring
```

There should be one Node Exporter pod per Kind node.

Check service:

```bash
kubectl get svc -n monitoring
```

---

# 22. Kube-State-Metrics

Kube-State-Metrics exposes Kubernetes object state, including:

- Pod state
- Deployment state
- Namespace information
- Node information
- Replica information

Install:

```bash
helm install kube-state-metrics \
  prometheus-community/kube-state-metrics \
  -n monitoring
```

Verify:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

The service exposes metrics on port:

```text
8080
```

---

# 23. Kubelet / cAdvisor Metrics

A separate cAdvisor DaemonSet was not required.

The Kubernetes kubelet exposes cAdvisor metrics through:

```text
/api/v1/nodes/<node-name>/proxy/metrics/cadvisor
```

These metrics contain container resource information such as:

- CPU
- Memory
- Network
- Filesystem

The OpenTelemetry Collector was configured to access this endpoint.

---

# 24. OpenTelemetry Collector for Metrics and Traces

The OpenTelemetry Collector is the central collection component for metrics and traces.

It performs:

```text
Receive
   ↓
Process
   ↓
Export
```

The collector used:

```text
otel/opentelemetry-collector-contrib:0.160.0
```

It contained:

- Prometheus receiver
- OTLP receiver
- Batch processor
- ClickHouse exporter

---

# 25. RBAC for Kubernetes Discovery

The collector needs Kubernetes API permissions to discover nodes, pods, services, and endpoints.

Create:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-prometheus-discovery
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
      - endpoints
      - nodes
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-prometheus-discovery
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-prometheus-discovery
subjects:
  - kind: ServiceAccount
    name: otel-collector-opentelemetry-collector
    namespace: monitoring
```

Apply:

```bash
kubectl apply -f otel-rbac.yaml
```

---

# 26. RBAC for cAdvisor

The collector also needs access to node proxy endpoints.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-cadvisor
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-cadvisor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-cadvisor
subjects:
  - kind: ServiceAccount
    name: otel-collector-opentelemetry-collector
    namespace: monitoring
```

Verify:

```bash
kubectl auth can-i get nodes/proxy \
  --as=system:serviceaccount:monitoring:otel-collector-opentelemetry-collector
```

Expected:

```text
yes
```

---

# 27. Prometheus Scrape Configuration in OTel Collector

The collector used Kubernetes service discovery.

Node Exporter:

```yaml
- job_name: node-exporter
  scrape_interval: 30s
  kubernetes_sd_configs:
    - role: endpoints
  relabel_configs:
    - source_labels: [__meta_kubernetes_service_name]
      regex: node-exporter-prometheus-node-exporter
      action: keep
```

Kube-State-Metrics:

```yaml
- job_name: kube-state-metrics
  scrape_interval: 30s
  kubernetes_sd_configs:
    - role: endpoints
  relabel_configs:
    - source_labels: [__meta_kubernetes_service_name]
      regex: kube-state-metrics
      action: keep
```

cAdvisor:

```yaml
- job_name: cadvisor
  scrape_interval: 30s
  kubernetes_sd_configs:
    - role: node
  scheme: https
  metrics_path: /metrics/cadvisor

  tls_config:
    insecure_skip_verify: true

  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  relabel_configs:
    - target_label: __address__
      replacement: kubernetes.default.svc:443

    - source_labels: [__meta_kubernetes_node_name]
      target_label: __metrics_path__
      replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
```

> `insecure_skip_verify` was used for this local lab configuration. Production TLS verification should be configured according to the Kubernetes security model.

---

# 28. ClickHouse Exporter Configuration

The OpenTelemetry ClickHouse exporter was configured with:

```yaml
clickhouse:
  endpoint: tcp://clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000
  database: observability
  username: default
  create_schema: true
  async_insert: true
```

Important fields:

| Field | Meaning |
|---|---|
| endpoint | ClickHouse native protocol endpoint |
| database | Target database |
| username | ClickHouse user |
| create_schema | Allows exporter to create required tables |
| async_insert | Uses asynchronous ClickHouse inserts |

---

# 29. Metrics Pipeline

The metrics pipeline was:

```text
Node Exporter
      |
Kube-State-Metrics
      |
   cAdvisor
      |
      v
Prometheus Receiver
      |
    Batch
      |
ClickHouse Exporter
      |
      v
ClickHouse
```

Configuration:

```yaml
service:
  pipelines:
    metrics:
      receivers:
        - prometheus
      processors:
        - batch
      exporters:
        - clickhouse
```

---

# 30. Verify ClickHouse Metrics

Check tables:

```sql
SHOW TABLES FROM observability;
```

Expected metric tables include:

```text
otel_metrics_gauge
otel_metrics_sum
otel_metrics_histogram
otel_metrics_summary
otel_metrics_exponential_histogram
```

Example:

```sql
SELECT
    MetricName,
    count()
FROM observability.otel_metrics_gauge
GROUP BY MetricName
ORDER BY count() DESC
LIMIT 20;
```

Verify Kubernetes pod readiness:

```sql
SELECT
    MetricName,
    Attributes['namespace'] AS namespace,
    Attributes['pod'] AS pod,
    Attributes['phase'] AS phase,
    Value
FROM observability.otel_metrics_gauge
WHERE MetricName = 'kube_pod_status_ready'
LIMIT 20;
```

Verify container memory:

```sql
SELECT
    MetricName,
    Attributes['namespace'] AS namespace,
    Attributes['pod'] AS pod,
    Value
FROM observability.otel_metrics_gauge
WHERE MetricName = 'container_memory_working_set_bytes'
LIMIT 20;
```

---

# 31. Traces

The OpenTelemetry Collector also receives OTLP traces.

Receivers:

```yaml
otlp:
  protocols:
    grpc:
      endpoint: 0.0.0.0:4317
    http:
      endpoint: 0.0.0.0:4318
```

Trace pipeline:

```text
Test Application
      |
   OTLP HTTP
      |
   Collector
      |
    Batch
      |
ClickHouse Exporter
      |
      v
observability.otel_traces
```

Pipeline:

```yaml
service:
  pipelines:
    traces:
      receivers:
        - otlp
      processors:
        - batch
      exporters:
        - clickhouse
```

---

# 32. Generate a Test Trace

The test application used the Python OpenTelemetry SDK.

The OTLP HTTP endpoint was:

```text
http://otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4318/v1/traces
```

The generated trace used:

```text
service.name = clickhouse-trace-test
operation    = test-operation
```

Verify in ClickHouse:

```sql
SELECT
    ResourceAttributes['service.name'] AS service,
    SpanName,
    TraceId,
    SpanId,
    Timestamp
FROM observability.otel_traces
ORDER BY Timestamp DESC
LIMIT 20;
```

A successful test produced a trace similar to:

```text
clickhouse-trace-test
test-operation
```

---

# 33. Verify All ClickHouse Tables

```sql
SHOW TABLES FROM observability;
```

Expected tables include:

```text
kubernetes_logs
logs
otel_metrics_exponential_histogram
otel_metrics_gauge
otel_metrics_histogram
otel_metrics_sum
otel_metrics_summary
otel_traces
```

Temporary test tables may also exist during development. They can be removed after verification.

---

# 34. Expose ClickHouse to Grafana

Grafana was running on an AWS EC2 instance while the ClickHouse cluster was running locally in Kind.

A NodePort service was used to expose ClickHouse HTTP.

Example:

```text
Kind worker
    |
NodePort 30081
    |
ClickHouse HTTP 8123
```

Test locally:

```bash
curl http://172.19.0.4:30081/ping
```

Expected:

```text
Ok.
```

---

# 35. SSH Reverse Tunnel for ClickHouse

Grafana EC2:

```text
34.229.147.247
```

SSH user:

```text
ubuntu
```

Local SSH key:

```text
~/Downloads/k8skey.pem
```

Create the reverse tunnel:

```bash
cd ~/Downloads

ssh -o ExitOnForwardFailure=yes \
  -i k8skey.pem \
  -N \
  -R 127.0.0.1:8123:172.19.0.4:30081 \
  ubuntu@34.229.147.247
```

From the EC2 instance:

```bash
curl http://127.0.0.1:8123/ping
```

Expected:

```text
Ok.
```

> The reverse tunnel is a lab connectivity mechanism. It is not a production architecture.

---

# 36. Grafana ClickHouse Datasource

In Grafana, create a ClickHouse datasource.

The datasource used in the POC was:

```text
grafana-clickhouse-datasource-1
```

For the HTTP connection, the tunnel endpoint was:

```text
http://127.0.0.1:8123
```

The ClickHouse database was:

```text
observability
```

---

# 37. ClickHouse Grafana Dashboard

The dashboard was named:

```text
Clickhouse Observability
```

It contained the following sections.

## Logs

### Kubernetes Logs

```sql
SELECT
    timestamp,
    namespace,
    pod,
    container,
    node,
    message
FROM observability.kubernetes_logs
WHERE $__timeFilter(timestamp)
ORDER BY timestamp DESC
LIMIT 100;
```

### Log Volume

```sql
SELECT
    toStartOfMinute(timestamp) AS time,
    count() AS log_count
FROM observability.kubernetes_logs
WHERE $__timeFilter(timestamp)
GROUP BY time
ORDER BY time;
```

### Error Log Count

```sql
SELECT count() AS error_count
FROM observability.kubernetes_logs
WHERE $__timeFilter(timestamp)
  AND positionCaseInsensitive(message, 'ERROR') > 0;
```

### Logs by Namespace

```sql
SELECT
    namespace,
    count() AS log_count
FROM observability.kubernetes_logs
WHERE $__timeFilter(timestamp)
GROUP BY namespace
ORDER BY log_count DESC;
```

---

# 38. Host Resource Panels

The dashboard used ClickHouse metrics to display:

- CPU cores
- CPU utilization
- Total memory
- Memory utilization
- Storage
- Storage utilization
- Disk I/O
- Network throughput

For example, CPU metrics came from:

```text
node_cpu_seconds_total
```

Memory metrics used:

```text
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
```

---

# 39. Kubernetes Workload Panels

The dashboard included:

- Pods by namespace
- Pod status
- Pod details
- Pod CPU
- Pod memory
- Pod network
- Pod restart information

Kubernetes metrics came primarily from:

```text
kube_pod_info
kube_pod_status_phase
container_cpu_*
container_memory_*
```

---

# 40. Trace Panels

The trace dashboard included:

- Trace count
- Average trace duration
- Trace details

Example query:

```sql
SELECT
    Timestamp AS time,
    ResourceAttributes['service.name'] AS service,
    SpanName,
    TraceId,
    SpanId
FROM observability.otel_traces
WHERE $__timeFilter(Timestamp)
ORDER BY Timestamp DESC
LIMIT 100;
```

---

# 41. Data Volume and Storage

The dashboard also displayed the number of records stored by signal.

Logs:

```sql
SELECT count()
FROM observability.kubernetes_logs;
```

Metrics:

```sql
SELECT sum(rows)
FROM system.parts
WHERE active = 1
  AND database = 'observability'
  AND table LIKE 'otel_metrics_%';
```

Traces:

```sql
SELECT count()
FROM observability.otel_traces;
```

---

# 42. ClickHouse Physical Storage

To calculate active ClickHouse storage:

```sql
SELECT
    round(sum(bytes_on_disk) / 1024 / 1024, 2) AS storage_mb
FROM system.parts
WHERE active = 1;
```

For per-table storage:

```sql
SELECT
    database,
    table,
    round(sum(bytes_on_disk) / 1024 / 1024, 2) AS storage_mb
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY storage_mb DESC;
```

---

# 43. ClickHouse Compression

Compression can be calculated using `system.columns`.

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    round(
        sum(data_uncompressed_bytes)
        / nullIf(sum(data_compressed_bytes), 0),
        2
    ) AS compression_ratio
FROM system.columns
WHERE database = 'observability'
GROUP BY database, table
ORDER BY sum(data_compressed_bytes) DESC;
```

An observed POC snapshot showed an overall compression ratio of approximately:

```text
54.1 : 1
```

Individual tables showed different ratios.

For example:

```text
otel_metrics_gauge     ~57.7 : 1
otel_metrics_sum       ~55.7 : 1
kubernetes_logs        ~21.8 : 1
```

These are measurements from the POC dataset at that time. They are not universal ClickHouse compression guarantees.

---

# 44. ClickHouse Health Panels

## Running ClickHouse Pods

```sql
SELECT
    countDistinct(Attributes['uid']) AS running_pods
FROM observability.otel_metrics_gauge
WHERE MetricName = 'kube_pod_status_phase'
  AND Attributes['namespace'] = 'clickhouse'
  AND Attributes['phase'] = 'Running'
  AND Value = 1;
```

## ClickHouse Memory

```sql
SELECT
    max(memory_bytes) / 1024 / 1024 AS memory_mb
FROM
(
    SELECT
        Attributes['pod'] AS pod,
        Attributes['container'] AS container,
        argMax(Value, TimeUnix) AS memory_bytes
    FROM observability.otel_metrics_gauge
    WHERE MetricName = 'container_memory_working_set_bytes'
      AND Attributes['namespace'] = 'clickhouse'
      AND Attributes['pod'] LIKE 'clickhouse-clickhouse%'
      AND Attributes['container'] != ''
    GROUP BY pod, container
);
```

## Recent ClickHouse Queries

```sql
SELECT
    event_time AS Time,
    user AS User,
    query AS Query,
    query_duration_ms AS "Duration (ms)",
    read_rows AS "Read Rows",
    written_rows AS "Written Rows"
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 50;
```

## Query Errors

```sql
SELECT count() AS error_count
FROM system.query_log
WHERE type IN ('ExceptionWhileProcessing', 'ExceptionBeforeStart')
  AND event_time >= now() - INTERVAL 1 HOUR;
```

---

# 45. Important ClickHouse Troubleshooting

## Problem: ClickHouse Memory Limit Exceeded

Symptom:

```text
Code: 241
Memory limit exceeded
```

Cause:

The ClickHouse container had an insufficient memory limit.

Fix:

Increase:

```yaml
limits:
  memory: 2Gi
```

---

## Problem: ClickHouse DNS Lookup Timeout

Symptom:

```text
lookup clickhouse-...svc.cluster.local: i/o timeout
```

Actions used:

- Checked CoreDNS.
- Tested service DNS resolution.
- Changed operator pod DNS `ndots` behavior.
- Adjusted CoreDNS forwarding for the isolated lab.

---

## Problem: Fluent Bit Too Many Open Files

Symptom:

```text
errno=24 Too many open files
```

Fix:

```text
Inotify_Watcher false
```

The Kind host also required increasing inotify instances.

Temporary fix:

```bash
sudo sysctl -w fs.inotify.max_user_instances=1024
```

Persistent configuration:

```bash
echo 'fs.inotify.max_user_instances = 1024' \
  | sudo tee /etc/sysctl.d/99-kind-inotify.conf

sudo sysctl --system
```

---

# Part 2 — Traditional Observability Stack

# 46. Traditional Architecture

The traditional POC was deployed separately in:

```text
traditional-observability
```

Architecture:

```text
                 Kubernetes
                     |
       +-------------+-------------+
       |             |             |
   Exporters      Fluent Bit     OTel Collector
       |             |             |
    vmagent          |            Tempo
       |             |             |
       v             v             v
VictoriaMetrics    Loki          Tempo
       |             |             |
       +-------------+-------------+
                     |
                   Grafana
```

The traditional architecture uses specialized storage backends:

```text
Metrics → VictoriaMetrics
Logs    → Loki
Traces  → Tempo
```

---

# 47. Create Traditional Observability Namespace

```bash
kubectl create namespace traditional-observability
```

---

# 48. VictoriaMetrics

VictoriaMetrics is the metrics storage backend.

Add repository:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts
helm repo update
```

Install:

```bash
helm install victoria-metrics \
  vm/victoria-metrics-single \
  -n traditional-observability
```

Verify:

```bash
kubectl get pods -n traditional-observability
```

Expected:

```text
victoria-metrics-victoria-metrics-single-server-0
```

Check service:

```bash
kubectl get svc -n traditional-observability
```

VictoriaMetrics listens on:

```text
8428
```

---

# 49. vmagent

vmagent scrapes Prometheus-compatible metrics and forwards them to VictoriaMetrics.

Create `/tmp/vmagent-custom-values.yaml`:

```yaml
remoteWrite:
  - url: "http://victoria-metrics-victoria-metrics-single-server:8428/api/v1/write"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 300m
    memory: 256Mi

config:
  global:
    scrape_interval: 30s
    scrape_timeout: 10s

  scrape_configs:

    - job_name: node-exporter
      kubernetes_sd_configs:
        - role: endpoints
      relabel_configs:
        - source_labels: [__meta_kubernetes_service_name]
          regex: node-exporter-prometheus-node-exporter
          action: keep

    - job_name: kube-state-metrics
      kubernetes_sd_configs:
        - role: endpoints
      relabel_configs:
        - source_labels: [__meta_kubernetes_service_name]
          regex: kube-state-metrics
          action: keep

    - job_name: cadvisor
      kubernetes_sd_configs:
        - role: node
      scheme: https
      metrics_path: /metrics/cadvisor

      tls_config:
        insecure_skip_verify: true

      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

      relabel_configs:
        - target_label: __address__
          replacement: kubernetes.default.svc:443

        - source_labels: [__meta_kubernetes_node_name]
          target_label: __metrics_path__
          replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
```

Install:

```bash
helm install vmagent \
  vm/victoria-metrics-agent \
  -n traditional-observability \
  -f /tmp/vmagent-custom-values.yaml
```

---

# 50. vmagent RBAC

Create:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vmagent-cadvisor
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vmagent-cadvisor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vmagent-cadvisor
subjects:
  - kind: ServiceAccount
    name: vmagent-victoria-metrics-agent
    namespace: traditional-observability
```

Verify:

```bash
kubectl auth can-i get nodes/proxy \
  --as=system:serviceaccount:traditional-observability:vmagent-victoria-metrics-agent
```

Expected:

```text
yes
```

---

# 51. Verify VictoriaMetrics

Port-forward:

```bash
kubectl port-forward \
  -n traditional-observability \
  svc/victoria-metrics-victoria-metrics-single-server \
  8428:8428
```

Query:

```bash
curl -s \
  "http://127.0.0.1:8428/api/v1/query?query=up"
```

The POC returned 8 active targets:

```text
cAdvisor        3
kube-state      2
node-exporter   3
```

All were:

```text
up = 1
```

---

# 52. Loki

Loki is the traditional log backend.

An initial attempt used the newer Grafana Loki chart:

```text
grafana/loki
```

The configuration did not successfully produce the required single-binary deployment in this lab, so that release was removed.

The working setup used:

```text
grafana/loki-stack
chart: 2.10.3
```

The actual Loki image in the working installation was:

```text
grafana/loki:2.6.1
```

Create values:

```yaml
loki:
  persistence:
    enabled: true
    size: 5Gi

promtail:
  enabled: false

grafana:
  enabled: false

prometheus:
  enabled: false
```

Install:

```bash
helm install loki-stack \
  grafana/loki-stack \
  -n traditional-observability \
  -f /tmp/loki-stack-values.yaml
```

Verify:

```bash
kubectl get pods -n traditional-observability
```

Expected:

```text
loki-stack-0   1/1   Running
```

Service:

```bash
kubectl get svc -n traditional-observability
```

Loki listens on:

```text
3100
```

---

# 53. Loki Health Check

```bash
kubectl exec -n traditional-observability loki-stack-0 -- \
  wget -qO- http://localhost:3100/ready
```

Expected:

```text
ready
```

---

# 54. Traditional Fluent Bit → Loki

A second Fluent Bit release was used for the traditional stack.

Important configuration:

```yaml
config:
  service: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File /fluent-bit/etc/parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020
        Health_Check On

  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Exclude_Path /var/log/containers/fluent-bit-*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
        Inotify_Watcher false

  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

  outputs: |
    [OUTPUT]
        Name loki
        Match kube.*
        Host loki-stack.traditional-observability.svc.cluster.local
        Port 3100
        Labels job=fluent-bit,namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name'],container=$kubernetes['container_name']
        Line_Format json
```

The data path is:

```text
Kubernetes container logs
        ↓
Traditional Fluent Bit
        ↓
Loki
```

---

# 55. Verify Loki Labels

After logs are ingested:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/labels
```

The POC returned:

```json
{
  "status": "success",
  "data": [
    "container",
    "job",
    "namespace",
    "pod"
  ]
}
```

This confirms that Kubernetes log labels were being stored.

---

# 56. Query Loki Logs

Example LogQL:

```logql
{namespace=~".+"}
```

This returned real Kubernetes logs.

---

# 57. Loki Compatibility Limitation

Loki 2.6.1 was used because it was the working configuration for this lab.

An attempt to use a newer Loki image caused configuration compatibility problems.

Grafana's generic Loki datasource health check also produced a parse/compatibility error for the older Loki API.

However:

- Loki `/ready` worked.
- Loki label API worked.
- Real log queries worked.
- Grafana Explore could display logs.

Therefore, the limitation was documented rather than treating it as an ingestion failure.

---

# 58. Tempo

Tempo is the traditional trace backend.

Repository:

```text
grafana/tempo
```

The working chart was:

```text
1.24.4
```

Create values:

```yaml
tempo:
  reportingEnabled: false

  metricsGenerator:
    enabled: false

  storage:
    trace:
      backend: local
      local:
        path: /var/tempo

persistence:
  enabled: true
  size: 5Gi

service:
  type: ClusterIP
```

Install:

```bash
helm install tempo \
  grafana/tempo \
  -n traditional-observability \
  -f /tmp/tempo-values.yaml
```

Verify:

```bash
kubectl get pods -n traditional-observability
```

Expected:

```text
tempo-0   1/1   Running
```

---

# 59. Tempo Ports

Tempo exposed:

```text
3200  HTTP API
4317  OTLP gRPC
4318  OTLP HTTP
```

Health:

```bash
kubectl exec -n traditional-observability tempo-0 -- \
  wget -qO- http://localhost:3200/ready
```

Expected:

```text
ready
```

---

# 60. Traditional OpenTelemetry Collector

The traditional collector was configured only for traces.

Image:

```text
otel/opentelemetry-collector-contrib:0.160.0
```

Architecture:

```text
Application
    |
 OTLP 4317/4318
    |
OTel Collector
    |
 Tempo exporter
    |
    v
  Tempo
```

The collector used:

- OTLP receiver
- Batch processor
- Tempo exporter
- Traces pipeline

Tempo endpoint:

```text
tempo.traditional-observability.svc.cluster.local:4317
```

---

# 61. Generate a Traditional Test Trace

A test trace was generated with:

```text
service.name = traditional-trace-test
operation    = test-operation
```

The trace was sent to the traditional OpenTelemetry Collector.

Grafana Tempo Explore was able to display the trace.

Example TraceQL:

```text
{ resource.service.name = "traditional-trace-test" }
```

---

# 62. Expose Traditional Services with NodePorts

The traditional services were exposed using NodePorts.

| Service | NodePort | Backend |
|---|---:|---|
| loki-external | 32446 | Loki 3100 |
| vm-external | 31720 | VictoriaMetrics 8428 |
| tempo-external | 31000 | Tempo 3200 |

Check:

```bash
kubectl get svc -n traditional-observability
```

---

# 63. Test Traditional NodePorts

Using Kind worker IP:

```text
172.19.0.4
```

Loki:

```bash
curl http://172.19.0.4:32446/ready
```

Expected:

```text
ready
```

VictoriaMetrics:

```bash
curl http://172.19.0.4:31720/health
```

Expected:

```text
OK
```

Tempo:

```bash
curl http://172.19.0.4:31000/ready
```

Expected:

```text
ready
```

---

# 64. Reverse Tunnel for Traditional Stack

The Grafana EC2 instance needs access to the local Kind NodePorts.

Run from the laptop:

```bash
cd ~/Downloads

ssh -o ExitOnForwardFailure=yes \
  -i k8skey.pem \
  -N \
  -R 127.0.0.1:32446:172.19.0.4:32446 \
  -R 127.0.0.1:31720:172.19.0.4:31720 \
  -R 127.0.0.1:31000:172.19.0.4:31000 \
  ubuntu@34.229.147.247
```

From the Grafana EC2:

```bash
curl http://127.0.0.1:32446/ready
curl http://127.0.0.1:31720/health
curl http://127.0.0.1:31000/ready
```

Expected:

```text
ready
OK
ready
```

---

# 65. Grafana Datasources

The Grafana server contained these datasources:

| Datasource | Backend |
|---|---|
| grafana-clickhouse-datasource-1 | ClickHouse |
| loki-1 | Loki |
| VM-1 | VictoriaMetrics |
| tempo | Tempo |

The ClickHouse datasource reads the unified observability data.

The traditional datasources read from their specialized backends.

---

# 66. Comparison Dashboard

A separate Grafana dashboard was created:

```text
ClickHouse vs Traditional Observability
```

The dashboard compares the two architectures.

Panels created:

1. Log Volume: ClickHouse vs Loki
2. Metrics Availability: ClickHouse vs VictoriaMetrics
3. Trace Volume: ClickHouse vs Tempo
4. Storage Usage: ClickHouse vs Traditional Stack

---

# 67. Log Volume Comparison

ClickHouse query:

```sql
SELECT
    toStartOfMinute(timestamp) AS time,
    count() AS "ClickHouse"
FROM observability.kubernetes_logs
WHERE $__timeFilter(timestamp)
GROUP BY time
ORDER BY time;
```

Loki query:

```logql
sum(count_over_time({namespace=~".+"}[1m]))
```

The panel uses Grafana's Mixed datasource mode.

---

# 68. Metrics Comparison

ClickHouse:

```sql
SELECT
    toStartOfMinute(TimeUnix) AS time,
    uniq(MetricName) AS "ClickHouse"
FROM observability.otel_metrics_gauge
WHERE $__timeFilter(TimeUnix)
GROUP BY time
ORDER BY time;
```

VictoriaMetrics:

```promql
count({__name__=~".+"})
```

> These two queries measure different things: ClickHouse counts unique metric names in the selected table, while VictoriaMetrics counts active series. This difference should be kept in mind when interpreting the graph.

---

# 69. Trace Comparison

ClickHouse:

```sql
SELECT
    toStartOfMinute(Timestamp) AS time,
    count() AS "ClickHouse"
FROM observability.otel_traces
WHERE $__timeFilter(Timestamp)
GROUP BY time
ORDER BY time;
```

Tempo:

```text
{} | count_over_time()
```

Grafana Tempo range queries require an appropriate step. A 1-minute step was used in the POC.

---

# 70. Traditional Storage Inspection

Kubernetes PVCs:

```bash
kubectl get pvc -n traditional-observability
```

The POC had:

```text
VictoriaMetrics PVC  → 16Gi
Loki PVC              → 5Gi
Tempo PVC             → 5Gi
```

Actual data directory snapshots were also checked.

Loki:

```bash
kubectl exec -n traditional-observability loki-stack-0 -- \
  du -sh /data
```

Observed:

```text
10.4M
```

VictoriaMetrics:

```bash
kubectl exec -n traditional-observability \
  victoria-metrics-victoria-metrics-single-server-0 \
  -- du -sh /storage
```

Observed:

```text
29.4M
```

Tempo:

```bash
kubectl exec -n traditional-observability tempo-0 -- \
  du -sh /var/tempo
```

Observed:

```text
168.0K
```

These values are point-in-time usage snapshots and should not be interpreted as direct compression comparisons.

---

# 71. Why PVC Metrics Were Not Used

An attempt was made to query:

```promql
kubelet_volume_stats_used_bytes
```

However, the kubelet metrics endpoint in the Kind environment did not expose the expected volume usage gauges.

Checking:

```bash
kubectl get --raw \
  "/api/v1/nodes/kind-clickhouse-poc-worker/proxy/metrics" \
  | grep kubelet_volume_stats_used_bytes
```

returned no matching metric.

Only volume collection duration metrics were available.

Therefore, actual container data directories were inspected with `du -sh` instead.

---

# 72. Grafana EC2 Disk Issue

During the POC, the Grafana EC2 root filesystem became full.

The initial state was approximately:

```text
/dev/root   19G   19G   0   100%
```

The major usage was under:

```text
/var/log
```

The Grafana database itself was much smaller.

The EC2 root EBS volume was increased from:

```text
19 GiB → 40 GiB
```

After increasing the EBS volume, the Linux partition/filesystem also needs to be expanded.

For the ext4 root filesystem:

```bash
sudo resize2fs /dev/nvme0n1p1
```

Verify:

```bash
df -h /
```

> Increasing an AWS EBS volume does not automatically mean the filesystem has expanded. Both the block device and filesystem need to reflect the new size.

---

# 73. Traditional Stack Verification Checklist

Run:

```bash
kubectl get pods -n traditional-observability
```

All of the following should be healthy:

```text
VictoriaMetrics
vmagent
Loki
traditional Fluent Bit
Tempo
traditional OTel Collector
```

Check services:

```bash
kubectl get svc -n traditional-observability
```

Verify:

```text
Loki       → /ready
VictoriaMetrics → /health
Tempo      → /ready
```

Verify metrics:

```bash
curl -s \
  "http://127.0.0.1:8428/api/v1/query?query=up"
```

Verify Loki:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/labels
```

Verify Tempo:

```bash
kubectl exec -n traditional-observability tempo-0 -- \
  wget -qO- http://localhost:3200/ready
```

---

# 74. Complete ClickHouse Verification Checklist

## Kubernetes

```bash
kubectl get nodes
kubectl get pods -n clickhouse
kubectl get pods -n monitoring
kubectl get pods -n logging
```

## Keeper

```bash
kubectl get keepercluster -n clickhouse
```

## ClickHouse

```bash
kubectl get clickhousecluster -n clickhouse
```

## Services

```bash
kubectl get svc -n clickhouse
```

## Database

```sql
SHOW DATABASES;
```

## Tables

```sql
SHOW TABLES FROM observability;
```

## Logs

```sql
SELECT count()
FROM observability.kubernetes_logs;
```

## Metrics

```sql
SELECT count()
FROM observability.otel_metrics_gauge;
```

## Traces

```sql
SELECT count()
FROM observability.otel_traces;
```

---

# 75. Useful ClickHouse System Queries

Check databases:

```sql
SELECT name
FROM system.databases
ORDER BY name;
```

Check tables:

```sql
SELECT
    database,
    name
FROM system.tables
WHERE database = 'observability'
ORDER BY name;
```

Check parts:

```sql
SELECT
    database,
    table,
    count() AS parts,
    sum(rows) AS rows,
    sum(bytes_on_disk) AS bytes
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY bytes DESC;
```

Check running queries:

```sql
SELECT
    query_id,
    user,
    query,
    elapsed,
    read_rows
FROM system.processes;
```

Check recent queries:

```sql
SELECT
    event_time,
    type,
    query_duration_ms,
    read_rows,
    written_rows,
    query
FROM system.query_log
ORDER BY event_time DESC
LIMIT 20;
```

---

# 76. End-to-End Data Flow

## Logs

```text
Kubernetes container
       ↓
/var/log/containers/*.log
       ↓
Fluent Bit
       ↓
HTTP / ClickHouse 8123
       ↓
observability.kubernetes_logs
       ↓
Grafana
```

## Metrics

```text
Node Exporter
Kube-State-Metrics
Kubelet/cAdvisor
       ↓
OpenTelemetry Collector
       ↓
ClickHouse exporter
       ↓
observability.otel_metrics_*
       ↓
Grafana
```

## Traces

```text
Application
       ↓
OTLP
       ↓
OpenTelemetry Collector
       ↓
ClickHouse exporter
       ↓
observability.otel_traces
       ↓
Grafana
```

---

# 77. Traditional End-to-End Data Flow

## Logs

```text
Kubernetes
    ↓
Fluent Bit
    ↓
Loki
    ↓
Grafana
```

## Metrics

```text
Node Exporter
Kube-State-Metrics
Kubelet/cAdvisor
    ↓
vmagent
    ↓
VictoriaMetrics
    ↓
Grafana
```

## Traces

```text
Application
    ↓
OTLP
    ↓
OpenTelemetry Collector
    ↓
Tempo
    ↓
Grafana
```

---

# 78. Important Lab-Specific Notes

The following configurations were used specifically because this was a local Kind-based POC:

- Kind local-path storage.
- NodePort exposure.
- SSH reverse tunnels.
- `fs.inotify.max_user_instances=1024`.
- Fluent Bit `Inotify_Watcher false`.
- ClickHouse default user without a configured password.
- CoreDNS public DNS forwarding.
- `ndots=1` for the ClickHouse Operator.
- `insecure_skip_verify: true` for local cAdvisor scraping.
- Single ClickHouse shard and replica.
- Single Keeper replica.
- Local Tempo storage.
- Loki 2.6.1 compatibility configuration.

These settings should be reviewed before being used in production.

---

# 79. Reproduction Order for a Fresher

If the POC needs to be recreated from scratch, follow this order:

```text
1. Install Docker
2. Install kubectl
3. Install Kind
4. Install Helm
5. Create Kind cluster
6. Verify Kubernetes nodes
7. Create clickhouse namespace
8. Install cert-manager
9. Install ClickHouse Operator
10. Fix/verify operator DNS if required
11. Deploy ClickHouse Keeper
12. Deploy ClickHouse
13. Verify ClickHouse
14. Create observability database
15. Create sample MergeTree table
16. Deploy Fluent Bit
17. Create Kubernetes logs table
18. Verify logs in ClickHouse
19. Deploy Node Exporter
20. Deploy Kube-State-Metrics
21. Configure OTel Collector RBAC
22. Configure Prometheus receiver
23. Configure cAdvisor scraping
24. Configure ClickHouse exporter
25. Verify metrics in ClickHouse
26. Configure OTLP receiver
27. Generate test trace
28. Verify traces in ClickHouse
29. Expose ClickHouse through NodePort
30. Create SSH reverse tunnel
31. Configure Grafana ClickHouse datasource
32. Build ClickHouse dashboard

Traditional stack:

33. Create traditional-observability namespace
34. Install VictoriaMetrics
35. Install vmagent
36. Configure vmagent RBAC
37. Verify metrics
38. Install Loki
39. Deploy traditional Fluent Bit
40. Verify Loki logs
41. Install Tempo
42. Deploy traditional OTel Collector
43. Generate test trace
44. Verify Tempo
45. Create NodePorts
46. Create SSH reverse tunnels
47. Add Loki datasource
48. Add VictoriaMetrics datasource
49. Add Tempo datasource
50. Build comparison dashboard
51. Perform final end-to-end verification
```

---

# 80. Final State

At the end of the POC, the local Kubernetes environment contained two independently operating observability architectures.

### ClickHouse architecture

```text
Logs    → Fluent Bit → ClickHouse
Metrics → OTel Collector → ClickHouse
Traces  → OTel Collector → ClickHouse
```

### Traditional architecture

```text
Logs    → Fluent Bit → Loki
Metrics → vmagent → VictoriaMetrics
Traces  → OTel Collector → Tempo
```

Grafana was used as the common visualization layer.

This setup provides a reproducible environment for operating, troubleshooting, querying, and comparing the two observability architectures using the same Kubernetes environment.
