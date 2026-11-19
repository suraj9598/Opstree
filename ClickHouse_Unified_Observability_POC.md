# ClickHouse Unified Observability POC

## 1. Project Overview

The objective of this Proof of Concept (POC) is to evaluate **ClickHouse as a centralized observability backend** for storing and querying three major observability signals:

- Logs
- Metrics
- Traces

The POC uses a local Kubernetes environment powered by Kind. Three independent ingestion pipelines will collect logs, metrics, and traces and send them to ClickHouse for centralized storage and analysis.

Grafana will be used to visualize the collected observability data.

## 2. POC Objectives

The primary objectives are:

- Evaluate ClickHouse as a centralized storage backend for logs, metrics, and traces.
- Implement independent ingestion pipelines for all three observability signals.
- Collect Kubernetes container logs using Fluent Bit.
- Scrape infrastructure and application metrics using the OpenTelemetry Collector Prometheus Receiver.
- Collect distributed traces using OpenTelemetry instrumentation.
- Store all collected signals in ClickHouse.
- Visualize logs, metrics, and traces through Grafana.
- Explore correlation between logs and traces using trace identifiers.
- Evaluate the feasibility of using ClickHouse for unified observability.

## 3. POC Scope

### 3.1 In Scope

**1. Kubernetes Environment**
- Create a separate Kind cluster.
- Deploy and manage POC components inside this cluster.
- Keep the implementation suitable for available local system resources.

**2. Log Ingestion**
- Deploy Fluent Bit as a Kubernetes DaemonSet.
- Collect container logs from Kubernetes workloads.
- Send logs directly from Fluent Bit to ClickHouse.
- Validate log ingestion, storage, and querying.

**3. Metrics Collection**
- Deploy Node Exporter, cAdvisor, and kube-state-metrics.
- Use the OpenTelemetry Collector Prometheus Receiver to scrape metrics.
- Export collected metrics to ClickHouse.
- Validate metrics ingestion and querying.

**4. Distributed Tracing**
- Develop two demo microservices: Order API and Payment Service.
- Instrument applications using the OpenTelemetry SDK.
- Send traces to the OpenTelemetry Collector through OTLP.
- Export traces from the Collector to ClickHouse.
- Validate trace ingestion and querying.

**5. Centralized Storage**
- Use ClickHouse as the common storage backend.
- Configure the required databases and tables for each signal.
- Validate data availability and query performance.

**6. Visualization and Correlation**
- Deploy Grafana.
- Configure ClickHouse as a Grafana datasource.
- Create dashboards for logs, metrics, and traces.
- Explore correlation between logs and traces using `trace_id`, `span_id`, and service information.

### 3.2 Out of Scope

The following items are not included in the current POC:

- Production-grade High Availability (HA) and multi-node ClickHouse deployment.
- Multi-cluster Kubernetes monitoring.
- Production disaster recovery and backup strategy.
- Deployment of a Prometheus Server.
- Sending logs through the OpenTelemetry Collector.
- Replacing or modifying existing production observability infrastructure.
- Production-scale load testing and benchmarking.

### 3.3 Expected Outcome

At the end of the POC, we aim to demonstrate that:

- Logs, metrics, and traces can be collected through independent ingestion pipelines.
- ClickHouse can store all three observability signals in a centralized backend.
- Collected data can be queried successfully from ClickHouse.
- Grafana can visualize the collected observability data.
- Logs and traces can be correlated using trace identifiers.
- The architecture can serve as a foundation for evaluating a future production deployment.

**POC Limitation:** This is a local functional validation, not a production-readiness or HA certification exercise.

## 4. Proposed Architecture

The POC will have three independent ingestion pipelines.

### 4.1 Logs Pipeline

```text
Kubernetes Workloads
        |
        v
Container Logs
        |
        v
Fluent Bit (DaemonSet)
        |
        v
ClickHouse
```

**Workflow:**
- Kubernetes workloads generate container logs.
- Fluent Bit collects logs from Kubernetes nodes.
- Fluent Bit sends logs directly to ClickHouse.
- Logs are stored and queried in ClickHouse.

The OpenTelemetry Collector is not involved in the logs ingestion pipeline.

### 4.2 Metrics Pipeline

```text
Node Exporter
cAdvisor
kube-state-metrics
Application /metrics
        |
        v
OpenTelemetry Collector
(Prometheus Receiver)
        |
        v
ClickHouse Exporter
        |
        v
ClickHouse
```

**Workflow:**
- Exporters expose metrics through their Prometheus-compatible endpoints.
- The OpenTelemetry Collector Prometheus Receiver scrapes these endpoints.
- The Collector exports the collected metrics to ClickHouse.
- Metrics are stored and queried in ClickHouse.

No separate Prometheus Server will be deployed. The OpenTelemetry Collector will perform the scraping.

### 4.3 Distributed Tracing Pipeline

```text
       Order API
            |
            v
   Payment Service
            |
            v
 OpenTelemetry SDK
            |
            v
       OTLP
            |
            v
 OpenTelemetry Collector
       (OTLP Receiver)
            |
            v
     ClickHouse Exporter
            |
            v
        ClickHouse
```

**Workflow:**
- Order API and Payment Service generate distributed traces.
- Applications are instrumented using the OpenTelemetry SDK.
- Traces are sent to the OpenTelemetry Collector through OTLP.
- The Collector exports traces to ClickHouse.
- Traces are stored and queried in ClickHouse.

### 4.4 Visualization Layer

```text
             ClickHouse
                  |
                  v
               Grafana
                  |
          +-------+-------+
          |       |       |
         Logs   Metrics  Traces
```

Grafana will be used to query and visualize the data stored in ClickHouse.

### 4.5 Pipeline Summary

| Signal | Collection Component | Processing / Scraping | Destination |
|---|---|---|---|
| Logs | Fluent Bit | Direct collection and forwarding | ClickHouse |
| Metrics | Prometheus Exporters | OTel Collector Prometheus Receiver | ClickHouse |
| Traces | OpenTelemetry SDK | OTel Collector OTLP Receiver | ClickHouse |

**Important:** The OpenTelemetry Collector is used for metrics and traces only. Fluent Bit independently handles the logs pipeline.

## 5. Technology Stack

| Component | Technology | Purpose |
|---|---|---|
| Kubernetes | Kind | Local Kubernetes environment |
| Container Runtime | Docker | Container execution |
| Package Manager | Helm | Kubernetes application deployment |
| Kubernetes Controller | ClickHouse Operator | Manage ClickHouse and Keeper resources |
| Coordination | ClickHouse Keeper | Distributed coordination for ClickHouse |
| Storage Backend | ClickHouse | Centralized observability storage |
| Log Collection | Fluent Bit | Kubernetes container log collection |
| Metrics Exporter | Node Exporter | Node-level metrics |
| Metrics Exporter | cAdvisor | Container-level metrics |
| Metrics Exporter | kube-state-metrics | Kubernetes object and state metrics |
| Metrics Scraping | OTel Collector Prometheus Receiver | Scrape Prometheus-compatible metrics |
| Application Instrumentation | OpenTelemetry SDK | Generate distributed traces |
| Trace Transport | OTLP | Send traces to Collector |
| Telemetry Collector | OpenTelemetry Collector | Collect metrics and traces |
| Visualization | Grafana | Dashboards and visualization |

## 6. System Specifications

### 6.1 Laptop Specifications

| Component | Details |
|---|---|
| Operating System | Ubuntu 22.04.5 LTS |
| Laptop | Dell Latitude 3420 |
| Processor | Intel Core i5-1135G7 |
| Logical CPUs | 8 |
| Total RAM | 15 GiB |
| Swap | 2 GiB |
| Disk Capacity | 468 GB |
| Initially Available Disk Space | Approximately 368 GB |

### 6.2 Installed Tools

| Tool | Version |
|---|---|
| Docker | 29.7.2 |
| Kind | v0.33.0 |
| Helm | v4.3.0 |
| System kubectl | v1.29.15 |
| Project-local kubectl | v1.37.1 |
| ClickHouse Operator Helm Chart | 0.0.8 |
| ClickHouse Server | 26.10.4.207 |

## 7. Project Setup and Directory Structure

### 7.1 Project Workspace

```text
~/clickhouse-unified-poc
```

### 7.2 Directory Structure

```text
clickhouse-unified-poc/
│
├── application/
│
├── bin/
│   └── kubectl
│
├── configs/
│
├── docs/
│
├── helm/
│
├── kubernetes/
│
└── README.md
```

### 7.3 Directory Purpose

| Directory | Purpose |
|---|---|
| application | Application code for demo microservices |
| bin | Project-specific command-line binaries |
| configs | Configuration files |
| docs | Technical documentation |
| helm | Helm-related configurations |
| kubernetes | Kubernetes manifests |
| README.md | Project documentation |

## 8. Project-local kubectl Installation

The system already had kubectl v1.29.15 installed.

To avoid modifying the existing system-level kubectl installation, a separate project-local kubectl v1.37.1 binary was installed.

### 8.1 Installation Steps

```bash
cd ~/clickhouse-unified-poc

mkdir -p bin

curl -LO https://dl.k8s.io/release/v1.37.1/bin/linux/amd64/kubectl

curl -LO https://dl.k8s.io/release/v1.37.1/bin/linux/amd64/kubectl.sha256

echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

install -m 0755 kubectl bin/kubectl
```

### 8.2 Verification

```bash
./bin/kubectl version --client
```

Verified client version:

```text
Client Version: v1.37.1
Kustomize Version: v5.8.1
```

Checksum verification:

```text
kubectl: OK
```

The project-local kubectl installation was completed successfully without changing the system-level kubectl version.

---

# PHASE 2: ClickHouse Installation and Kubernetes Deployment

## 9. Phase 2 Overview

The objective of Phase 2 was to deploy ClickHouse inside the newly created Kind Kubernetes cluster.

Instead of running ClickHouse as a standalone container, we deployed it using the **official ClickHouse Operator and Kubernetes Custom Resources**.

This approach allows Kubernetes to manage the ClickHouse deployment through declarative configuration.

The following components were deployed:

1. cert-manager
2. ClickHouse Operator
3. ClickHouse Operator Custom Resource Definitions (CRDs)
4. ClickHouse Keeper
5. ClickHouse Server
6. Persistent Volume Claims
7. Kubernetes Services and Endpoints

At the end of this phase, ClickHouse was successfully running and its native client connection was validated.

## 10. Understanding ClickHouse

### 10.1 What is ClickHouse?

ClickHouse is an open-source, column-oriented analytical database management system (OLAP).

It is designed to process large amounts of data and execute analytical queries efficiently.

Unlike traditional row-oriented databases, ClickHouse stores data in a columnar format.

This makes it particularly useful for analytical workloads where queries frequently read only a few columns from a large number of records.

### 10.2 OLTP vs OLAP

**OLTP (Online Transaction Processing)**

OLTP databases are designed for transactional workloads.

Examples:
- Creating user accounts.
- Updating customer details.
- Processing orders.
- Performing banking transactions.

These operations generally work with individual records and require frequent inserts and updates.

**OLAP (Online Analytical Processing)**

OLAP databases are designed for analytical workloads.

Examples:
- Counting application errors.
- Calculating average response times.
- Finding the number of requests received per minute.
- Analyzing CPU and memory usage.
- Identifying services generating the highest number of errors.

ClickHouse is designed primarily for OLAP workloads.

### 10.3 What is a Columnar Database?

A traditional row-oriented database stores records row by row.

For example:

```text
ID | Name  | City    | Age
---|-------|---------|----
1  | Rahul | Delhi   | 25
2  | Amit  | Mumbai  | 28
3  | Ravi  | Noida   | 30
```

Conceptually, the data is stored like this:

```text
Row 1: 1, Rahul, Delhi, 25
Row 2: 2, Amit, Mumbai, 28
Row 3: 3, Ravi, Noida, 30
```

In a columnar database, values are organized by column:

```text
ID:
1, 2, 3

Name:
Rahul, Amit, Ravi

City:
Delhi, Mumbai, Noida

Age:
25, 28, 30
```

**Why is columnar storage useful?**

Suppose we want to calculate the average age of all users.

```sql
SELECT AVG(Age)
FROM users;
```

The database primarily needs to read the `Age` column instead of reading every column of every row.

This can reduce unnecessary disk reads and improve analytical query performance.

### 10.4 Why ClickHouse for Observability?

Observability systems generate large volumes of data.

The three primary observability signals are:

| Signal | Description | Example |
|---|---|---|
| Logs | Records of events generated by applications and infrastructure | Application error, request failed |
| Metrics | Numerical measurements collected over time | CPU usage, memory usage, request count |
| Traces | Records of requests flowing across multiple services | Order API calling Payment Service |

ClickHouse can store and analyze these different types of data.

For this POC, the objective is to use one common backend for logs, metrics, and traces.

Some expected analytical queries include:

```sql
SELECT count()
FROM observability.otel_logs;
```

```sql
SELECT count()
FROM observability.otel_traces;
```

```sql
SELECT count()
FROM observability.otel_metrics_gauge;
```

These are example queries for the planned observability schema. The database and tables will be created in a subsequent implementation step.

### 10.5 Important ClickHouse Features

**1. Columnar Storage**

Data is stored column by column, making analytical queries efficient when only selected columns are required.

**2. Data Compression**

Similar values are stored together, allowing ClickHouse to apply compression techniques efficiently.

This can reduce storage requirements for large analytical datasets.

**3. SQL Support**

ClickHouse supports SQL-like queries for filtering, aggregation, grouping, sorting, and analysis.

Example:

```sql
SELECT
    count()
FROM system.tables;
```

**4. High-Performance Aggregations**

ClickHouse is designed for analytical operations such as:

- COUNT()
- SUM()
- AVG()
- MIN()
- MAX()
- GROUP BY

**5. MergeTree Family**

MergeTree is one of the most important table-engine families in ClickHouse.

It provides capabilities such as:
- Efficient data insertion.
- Sorted data storage.
- Data parts and background merging.
- Partitioning.
- Primary-key-based data organization.

For observability workloads, MergeTree-family engines are commonly used to store time-series-like logs, metrics, and traces.

**6. Distributed Query Processing**

ClickHouse supports distributed deployments across multiple servers and shards.

This allows workloads to be distributed across multiple machines when scaling beyond a single node.

**Important:** Our current POC uses one shard and one replica. Distributed and production HA capabilities have not been validated.

## 11. Understanding ClickHouse Operator

### 11.1 What is ClickHouse Operator?

ClickHouse Operator is a Kubernetes controller that automates the deployment and management of ClickHouse clusters.

Normally, deploying and managing ClickHouse manually would require creating and maintaining:

- StatefulSets.
- Services.
- Persistent Volume Claims.
- Configuration files.
- User configurations.
- Cluster coordination settings.
- Pod lifecycle and readiness.

The ClickHouse Operator automates much of this work.

We define the desired ClickHouse configuration using Kubernetes Custom Resources, and the Operator reconciles the actual Kubernetes resources to match that configuration.

### 11.2 How ClickHouse Operator Works

```text
          Kubernetes API
                |
                v
        ClickHouse Custom
            Resource
                |
                v
        ClickHouse Operator
                |
        +-------+-------+
        |       |       |
        v       v       v
    StatefulSet Service  PVC
        |
        v
   ClickHouse Pod
        |
        v
   ClickHouse Server
```

**Workflow:**

1. We create a ClickHouseCluster Custom Resource.
2. Kubernetes API stores the desired configuration.
3. ClickHouse Operator watches the Custom Resource.
4. The Operator validates the configuration.
5. The Operator creates or updates the required Kubernetes resources.
6. ClickHouse Server starts.
7. The Operator continuously checks the cluster state and reconciles differences.

### 11.3 What is a CRD?

CRD stands for Custom Resource Definition.

Kubernetes provides built-in resources such as:

- Pod
- Deployment
- StatefulSet
- Service
- ConfigMap

A CRD allows an operator to introduce additional resource types into Kubernetes.

The ClickHouse Operator provides these important CRDs:

```text
clickhouseclusters.clickhouse.com
keeperclusters.clickhouse.com
```

**ClickHouseCluster:**

Defines the desired ClickHouse cluster configuration, including:

- Number of shards.
- Number of replicas.
- Container image.
- CPU and memory resources.
- Persistent storage.
- Keeper reference.
- ClickHouse settings.

**KeeperCluster:**

Defines the ClickHouse Keeper deployment, including:

- Number of Keeper replicas.
- Container image.
- Resource limits.
- Persistent storage.
- Keeper settings.

We verified the CRDs using:

```bash
kubectl get crds | grep clickhouse
```

Verified resources:

```text
clickhouseclusters.clickhouse.com
keeperclusters.clickhouse.com
```

## 12. Understanding ClickHouse Keeper

### 12.1 What is ClickHouse Keeper?

ClickHouse Keeper is a coordination service used by ClickHouse for distributed operations.

It maintains coordination information and helps ClickHouse replicas coordinate their activities.

Some of its responsibilities include:

- Replica coordination.
- Maintaining cluster metadata.
- Tracking replica state.
- Coordinating replicated table operations.
- Maintaining distributed coordination information.

ClickHouse Keeper implements the ZooKeeper-compatible coordination protocol.

### 12.2 Why Do We Need Keeper?

In a ClickHouse deployment with replicated tables, multiple ClickHouse replicas need to coordinate their operations.

For example:

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

Keeper maintains coordination information that helps replicas remain aware of their state.

Without a coordination mechanism, managing replicated operations becomes difficult.

### 12.3 Keeper in Our POC

We deployed a separate ClickHouse Keeper resource.

Current configuration:

| Parameter | Value |
|---|---|
| Keeper Resource | clickhouse-keeper |
| Namespace | clickhouse |
| Replicas | 1 |
| Container Image | docker.io/clickhouse/clickhouse-keeper:latest |
| CPU Request | 100m |
| CPU Limit | 500m |
| Memory Request | 256Mi |
| Memory Limit | 512Mi |
| Storage | 2Gi |
| Storage Class | standard |
| Access Mode | ReadWriteOnce |

Keeper pod:

```text
clickhouse-keeper-keeper-0-0
```

### 12.4 Why Separate Keeper and ClickHouse Storage?

We use separate Persistent Volume Claims for ClickHouse Server and ClickHouse Keeper.

**ClickHouse storage:**

Stores database data, table data, metadata, and related server files.

**Keeper storage:**

Stores Keeper's coordination state and related persistent data.

Keeping these storage volumes separate provides:

- Independent storage management.
- Separate storage capacity allocation.
- Independent persistence for database data and coordination state.
- Easier troubleshooting and storage inspection.

Our current allocation is:

| Component | Storage |
|---|---|
| ClickHouse | 5Gi |
| ClickHouse Keeper | 2Gi |
| Total Requested Storage | 7Gi |

**Important:** Separate storage volumes do not provide high availability by themselves. Our current deployment still has only one ClickHouse replica and one Keeper replica.

## 13. Phase 2 Implementation

### 13.1 Kubernetes Cluster

A separate local Kind cluster was created for this POC.

Cluster details:

| Parameter | Value |
|---|---|
| Cluster Name | clickhouse-unified-poc |
| Kubernetes Version | v1.37.0 |
| Node Count | 1 |
| Node Type | Control Plane |
| Environment | Local Kind |

The cluster is separate from other existing Kubernetes environments.

### 13.2 Kubernetes Namespaces

The following namespaces were created for organizing the POC components:

| Namespace | Purpose |
|---|---|
| applications | Demo applications |
| clickhouse | ClickHouse Server and Keeper |
| logging | Log collection components |
| metrics | Metrics exporters |
| monitoring | Monitoring and visualization components |
| opentelemetry | OpenTelemetry Collector |

Namespace verification:

```bash
kubectl get namespaces
```

The POC namespaces were verified in Active status.

### 13.3 cert-manager Installation

cert-manager was deployed as a prerequisite for the ClickHouse Operator installation.

**What is cert-manager?**

cert-manager is a Kubernetes certificate management controller.

It automates certificate-related operations, such as:

- Certificate issuance.
- Certificate renewal.
- Managing Kubernetes Certificate resources.
- Working with issuers and certificate authorities.

The ClickHouse Operator deployment uses certificate resources for its webhook and metrics-related TLS configuration.

We verified the cert-manager deployment through its Kubernetes pods.

Verification:

```bash
kubectl get pods -n cert-manager
```

The cert-manager components were running successfully.

### 13.4 ClickHouse Operator Installation

After cert-manager was deployed, the ClickHouse Operator was installed using Helm.

Helm was used to manage the Kubernetes application deployment.

Installed Operator details:

| Parameter | Value |
|---|---|
| Helm Chart | clickhouse-operator-helm |
| Chart Version | 0.0.8 |
| Namespace | clickhouse-operator-system |
| Container Image | ghcr.io/clickhouse/clickhouse-operator:v0.0.8 |
| Controller Deployment | clickhouse-operator-controller-manager |

The Operator pod was verified using:

```bash
kubectl get pods -n clickhouse-operator-system
```

Expected component:

```text
clickhouse-operator-controller-manager
```

The Operator was running successfully.

### 13.5 CRD Verification

After Operator installation, we verified that the required Custom Resource Definitions were registered.

Command:

```bash
kubectl get crds | grep clickhouse
```

Output:

```text
clickhouseclusters.clickhouse.com
keeperclusters.clickhouse.com
```

We also inspected the CRD specifications using:

```bash
kubectl explain clickhouseclusters.clickhouse.com.spec
```

```bash
kubectl explain keeperclusters.clickhouse.com.spec
```

This helped us understand which configuration fields are supported by the installed Operator version.

### 13.6 ClickHouse Keeper Deployment

Before deploying the ClickHouse Server, we created the ClickHouse Keeper resource.

Keeper configuration:

| Parameter | Configured Value |
|---|---|
| Name | clickhouse-keeper |
| Namespace | clickhouse |
| Replicas | 1 |
| Image | docker.io/clickhouse/clickhouse-keeper:latest |
| CPU Request | 100m |
| CPU Limit | 500m |
| Memory Request | 256Mi |
| Memory Limit | 512Mi |
| Storage Request | 2Gi |
| Storage Class | standard |
| Access Mode | ReadWriteOnce |
| Cluster Domain | cluster.local |
| TLS | Disabled |

The ClickHouse Operator created the required Keeper Kubernetes resources.

Verification:

```bash
kubectl get pods -n clickhouse
```

Keeper pod:

```text
clickhouse-keeper-keeper-0-0
```

Keeper cluster verification:

```bash
kubectl get keeperclusters.clickhouse.com -n clickhouse
```

Keeper reached the Ready state.

### 13.7 ClickHouse Server Deployment

After Keeper became ready, we deployed the ClickHouse Server using a ClickHouseCluster Custom Resource.

This is the main database component of our POC.

**ClickHouse Cluster Configuration**

| Parameter | Configured Value |
|---|---|
| Cluster Name | clickhouse |
| Namespace | clickhouse |
| Shards | 1 |
| Replicas | 1 |
| Image Repository | docker.io/clickhouse/clickhouse-server |
| Image Tag | latest |
| CPU Request | 250m |
| CPU Limit | 1 |
| Memory Request | 512Mi |
| Memory Limit | 1Gi |
| Storage Request | 5Gi |
| Storage Class | standard |
| Access Mode | ReadWriteOnce |
| Keeper Reference | clickhouse-keeper |
| Cluster Domain | cluster.local |
| Database Sync | Enabled |
| TLS | Disabled |

**Understanding Shards and Replicas**

A shard represents a portion of the data distributed across a ClickHouse cluster.

A replica is another copy of a shard's data.

Example of a larger cluster:

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

Our current configuration is:

```text
Shards:   1
Replicas: 1
```

This means the current deployment contains one shard with one ClickHouse replica.

It is suitable for validating the installation and basic functionality, but it does not provide replica-level redundancy.

### 13.8 Persistent Storage Configuration

ClickHouse data must persist even when the ClickHouse Pod is recreated.

For this reason, we configured persistent storage through a Persistent Volume Claim.

ClickHouse PVC configuration:

| Parameter | Value |
|---|---|
| Storage Request | 5Gi |
| Storage Class | standard |
| Access Mode | ReadWriteOnce |

Keeper PVC configuration:

| Parameter | Value |
|---|---|
| Storage Request | 2Gi |
| Storage Class | standard |
| Access Mode | ReadWriteOnce |

**What is a PVC?**

A Persistent Volume Claim is a Kubernetes request for persistent storage.

The application requests storage through a PVC, and Kubernetes binds it to an appropriate Persistent Volume.

In our Kind cluster, the `standard` StorageClass is used for dynamic local-path provisioning.

Verification:

```bash
kubectl get pvc -n clickhouse
```

Expected storage claims include:

```text
clickhouse-storage-volume-clickhouse-clickhouse-0-0-0
```

and the Keeper storage claim.

The PVCs were verified as Bound during deployment.

### 13.9 Kubernetes Services

The ClickHouse Operator created Kubernetes Services to provide network access to ClickHouse and Keeper.

We verified them using:

```bash
kubectl get svc -n clickhouse
```

The following services were created:

| Service | Type | Ports | Purpose |
|---|---|---|---|
| clickhouse-clickhouse-headless | ClusterIP (Headless) | 8123, 9363, 9000 | ClickHouse client and metrics access |
| clickhouse-clickhouse-internal-0-0 | ClusterIP (Headless) | 9009, 9001, 9002 | Internal replica communication |
| clickhouse-keeper-keeper-headless | ClusterIP (Headless) | 9234, 9090, 2181 | Keeper communication |

**Important ClickHouse Ports**

| Port | Purpose |
|---|---|
| 8123 | HTTP interface |
| 9000 | Native TCP interface |
| 9001 | Replica-specific native TCP endpoint in this deployment |
| 9002 | Internal ClickHouse endpoint |
| 9009 | Interserver HTTP communication |
| 9363 | Prometheus-compatible metrics endpoint |

The exact ports and roles should always be interpreted according to the generated Operator configuration.

**Why Headless Services?**

A headless Service uses:

```yaml
clusterIP: None
```

Instead of providing a single virtual ClusterIP, Kubernetes DNS can return the addresses of the associated Pods.

This is useful for StatefulSet-based applications such as ClickHouse and Keeper, where individual Pod identities matter.

### 13.10 Kubernetes Endpoints Verification

We checked whether the Services were correctly associated with the running ClickHouse and Keeper Pods.

Command:

```bash
kubectl get svc,endpoints -n clickhouse
```

Verified endpoint mapping:

| Service | Endpoint |
|---|---|
| clickhouse-clickhouse-headless | 10.244.0.14:8123, 9363, 9000 |
| clickhouse-clickhouse-internal-0-0 | 10.244.0.14:9001, 9002, 9009 |
| clickhouse-keeper-keeper-headless | 10.244.0.11:2181, 9234, 9090 |

This confirmed that:

- ClickHouse Services were associated with the ClickHouse Pod.
- Keeper Service was associated with the Keeper Pod.
- Required service endpoints were populated.

The Kubernetes Endpoints API displayed a deprecation warning because Kubernetes 1.33 and later prefer EndpointSlices. The warning did not indicate a service failure.

## 14. Phase 2 Validation

After deployment, we performed multiple validation checks.

### 14.1 Pod Health Check

Command:

```bash
kubectl get pods -n clickhouse -o wide
```

Verified ClickHouse and Keeper Pods:

```text
NAME                                  READY   STATUS
clickhouse-clickhouse-0-0-0           1/1     Running
clickhouse-keeper-keeper-0-0          1/1     Running
```

The version-probe Job also completed successfully.

Operator verification:

```bash
kubectl get pods -n clickhouse-operator-system -o wide
```

The Operator controller was running with zero restarts.

### 14.2 ClickHouse Cluster Readiness

Command:

```bash
kubectl get clickhouseclusters.clickhouse.com -A
```

Verified output:

```text
NAMESPACE    NAME         READY   STATUS
clickhouse   clickhouse   True    All shards are ready
```

Replica information:

```text
READYREPLICAS: 1
REPLICAS:      1
VERSION:       26.10.4.207
```

**Interpretation:**

- `READY=True` means the Operator considers the cluster ready.
- `All shards are ready` means all configured shards have reached the ready state.
- `1/1` means the single configured replica is ready.

### 14.3 ClickHouse Client Connectivity

We tested connectivity using the ClickHouse native client inside the ClickHouse Pod.

Command:

```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- clickhouse-client
```

Verified output:

```text
ClickHouse client version 26.10.4.207
Connecting to localhost:9001 as user default.
Connected to ClickHouse server version 26.10.4.
```

This confirmed that the ClickHouse client could connect to the running server.

### 14.4 SQL Query Validation

After establishing the client connection, we executed a basic SQL query.

Query:

```sql
SELECT 1;
```

Verified output:

```text
┌─1─┐
│ 1 │
└───┘

1 row in set.
Elapsed: 0.001 sec.
```

This confirmed that:

- The ClickHouse server was accepting client connections.
- SQL queries could be executed.
- The query returned the expected result.

### 14.5 Database Verification

We executed:

```sql
SHOW DATABASES;
```

The current verified output contained:

```text
INFORMATION_SCHEMA
default
information_schema
system
```

**Important Implementation Status:**

The `observability` database was not present in this verification.

Therefore, ClickHouse Server installation and connectivity validation are completed, but the observability database and signal-specific tables still need to be created and validated.

### 14.6 Operator Reconciliation and DNS Investigation

During the initial deployment, the ClickHouse Operator reported a replica probe error:

```text
failed to probe replica

dial tcp:
lookup clickhouse-clickhouse-internal-0-0.clickhouse.svc.cluster.local:
i/o timeout
```

We investigated the Kubernetes DNS and Operator configuration.

The following checks were performed:

1. Checked ClickHouse Operator logs.
2. Verified ClickHouse and Keeper Pods.
3. Checked ClickHouse Services and Endpoints.
4. Inspected Operator Pod DNS policy.
5. Checked the DNS resolver configuration of a temporary debug Pod.
6. Inspected the Operator Pod description and events.
7. Rechecked ClickHouseCluster readiness.

The Operator Pod used:

```text
dnsPolicy: ClusterFirst
```

No custom `dnsConfig` was configured.

The temporary DNS debug Pod showed:

```text
nameserver 10.96.0.10
options ndots:5
```

Additional search domains were also present:

```text
netbird.selfhosted
tail5caa65.ts.net
```

The presence of these domains was observed, but it was not conclusively established as the root cause of the earlier DNS timeout.

The Operator image was minimal and did not contain the `cat` executable. Attempts to start BusyBox ephemeral debugger containers were rejected because the Operator Pod enforced `runAsNonRoot`.

The temporary `dns-debug` Pod was deleted after the investigation.

**Final observed status:**

```text
READY: True
STATUS: All shards are ready
READYREPLICAS: 1
REPLICAS: 1
```

The supplied latest Operator logs showed successful reconciliation without a DNS timeout in that log window.

**Conclusion:** The cluster recovered and reached the Ready state. The original DNS timeout's root cause remains unconfirmed.

## 15. Current ClickHouse Deployment Architecture

The deployed Phase 2 architecture is:

```text
                  Kind Cluster
                       |
                       v
                Kubernetes API
                       |
                       v
              ClickHouse Operator
                       |
              +--------+--------+
              |                 |
              v                 v
        KeeperCluster     ClickHouseCluster
              |                 |
              v                 v
       ClickHouse Keeper   ClickHouse Server
              |                 |
              v                 v
         Keeper PVC        ClickHouse PVC
              |                 |
              +--------+--------+
                       |
                       v
              ClickHouse Storage
```

The Operator manages both the Keeper and ClickHouse resources.

The ClickHouse Server uses Keeper for coordination, and both components have their own persistent storage.

## 16. Current Phase 2 Deployment Summary

| Component | Namespace | Status |
|---|---|---|
| Kind Cluster | N/A | Running |
| cert-manager | cert-manager | Installed |
| ClickHouse Operator | clickhouse-operator-system | Running |
| ClickHouse CRDs | Kubernetes API | Registered |
| Keeper Cluster | clickhouse | Ready |
| ClickHouse Cluster | clickhouse | Ready |
| ClickHouse Server | clickhouse | Running |
| ClickHouse PVC | clickhouse | Bound |
| Keeper PVC | clickhouse | Bound |
| ClickHouse Services | clickhouse | Available |
| ClickHouse Endpoints | clickhouse | Populated |
| Native Client Connectivity | clickhouse | Successful |
| Basic SQL Query | clickhouse | Successful |

## 17. Important Notes and Limitations

- The current POC uses one Kind control-plane node.
- ClickHouse is configured with one shard and one replica.
- Keeper is configured with one replica.
- This is a functional local deployment, not a production HA setup.
- Separate PVCs are configured for ClickHouse and Keeper.
- The ClickHouse image configuration uses the `latest` tag, while the deployed version was observed as `26.10.4.207`.
- TLS is currently disabled in the ClickHouse and Keeper configurations.
- The `observability` database has not yet been created.
- Logs, metrics, and traces ingestion have not yet been validated as part of this Phase 2 deployment.
- The DNS timeout recovered, but its root cause remains unconfirmed.

## 18. Implementation Roadmap

| No. | Task | Status |
|---|---|---|
| 1 | Create project workspace and directory structure | Completed |
| 2 | Record system specifications | Completed |
| 3 | Install project-local kubectl v1.37.1 | Completed |
| 4 | Prepare Kind cluster configuration | Completed |
| 5 | Create separate Kind cluster | Completed |
| 6 | Create Kubernetes namespaces | Completed |
| 7 | Deploy cert-manager | Completed |
| 8 | Install ClickHouse Operator | Completed |
| 9 | Verify ClickHouse Operator CRDs | Completed |
| 10 | Deploy ClickHouse Keeper | Completed |
| 11 | Deploy ClickHouse Server | Completed |
| 12 | Configure persistent storage | Completed |
| 13 | Verify ClickHouse Services and Endpoints | Completed |
| 14 | Validate ClickHouse client connectivity | Completed |
| 15 | Validate basic SQL query execution | Completed |
| 16 | Create observability database and tables | Pending |
| 17 | Configure Fluent Bit direct ClickHouse output | Pending |
| 18 | Validate log ingestion | Pending |
| 19 | Deploy Node Exporter, cAdvisor, and kube-state-metrics | Pending |
| 20 | Configure OTel Collector Prometheus Receiver | Pending |
| 21 | Validate metrics ingestion | Pending |
| 22 | Develop Order API and Payment Service | Pending |
| 23 | Instrument applications using OpenTelemetry SDK | Pending |
| 24 | Configure OTel Collector OTLP Receiver | Pending |
| 25 | Validate trace ingestion | Pending |
| 26 | Deploy Grafana | Pending |
| 27 | Configure ClickHouse datasource in Grafana | Pending |
| 28 | Create observability dashboards | Pending |
| 29 | Validate logs, metrics, and traces | Pending |
| 30 | Validate cross-signal correlation | Pending |
| 31 | Document POC results and findings | Pending |

## 19. Next Implementation Phase

The next phase will focus on preparing ClickHouse for observability data.

Planned activities:

1. Create the `observability` database.
2. Create the required log, metric, and trace tables.
3. Configure Fluent Bit to send Kubernetes container logs directly to ClickHouse.
4. Validate log ingestion using SQL queries.
5. Deploy Node Exporter, cAdvisor, and kube-state-metrics.
6. Configure the OpenTelemetry Collector Prometheus Receiver.
7. Export metrics to ClickHouse.
8. Develop and instrument demo microservices.
9. Send distributed traces through OTLP to the OpenTelemetry Collector.
10. Export traces to ClickHouse.
11. Configure Grafana to query ClickHouse.

## 20. Expected Final Outcome

The final outcome will be a working local Kubernetes-based unified observability POC in which:

- Fluent Bit sends Kubernetes logs directly to ClickHouse.
- OTel Collector scrapes metrics from Prometheus-compatible exporters and sends them to ClickHouse.
- OpenTelemetry-instrumented applications send traces through OTLP to the Collector, which exports them to ClickHouse.
- Grafana visualizes all three observability signals.
- Logs and traces can be correlated using trace identifiers.

This POC will help evaluate ClickHouse's capabilities as a centralized backend for unified observability.
