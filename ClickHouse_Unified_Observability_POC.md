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

---

# PHASE 3: ClickHouse Schema and Demo Microservices Development

## 21. Phase 3 Overview

The objective of Phase 3 was to prepare ClickHouse for observability data and develop two interconnected demo microservices to generate application logs, metrics, and distributed traces.

The following activities were performed:

1. Created the `observability` database in ClickHouse.
2. Created the required observability tables.
3. Developed the Order API microservice using Python FastAPI.
4. Developed the Payment Service microservice using Python FastAPI.
5. Implemented structured JSON logging.
6. Integrated OpenTelemetry SDK instrumentation.
7. Implemented distributed trace context propagation.
8. Added Prometheus-compatible metrics endpoints.
9. Built Docker images for both applications.
10. Loaded application images into the Kind cluster.
11. Deployed both applications into the Kubernetes `applications` namespace.
12. Validated inter-service communication.
13. Tested successful and failed payment scenarios.
14. Verified application logs and trace correlation.

At the end of this phase, both applications were running successfully inside Kubernetes, and distributed trace context propagation was verified across the services.

**Important:** Application log generation and trace context propagation are validated. Actual Fluent Bit log ingestion and OpenTelemetry Collector trace export are part of the subsequent implementation phases.

## 22. ClickHouse Database and Table Preparation

### 22.1 Observability Database

A dedicated database named `observability` was created in ClickHouse to store observability-related data.

Database creation:

```sql
CREATE DATABASE IF NOT EXISTS observability;
```

The database is intended to store:

- Application logs.
- Kubernetes logs.
- Metrics.
- Distributed traces.

### 22.2 Observability Table Preparation

The required ClickHouse tables were created for storing observability data.

The table structure is designed to support the different telemetry signals and their associated attributes.

The observability schema is intended to support:

| Signal | Data |
|---|---|
| Logs | Timestamp, service name, severity, message, trace ID, span ID |
| Metrics | Metric name, timestamp, value, resource attributes |
| Traces | Trace ID, span ID, parent span ID, service name, duration |

### 22.3 Why Separate Tables?

Different observability signals have different data structures and query requirements.

For example:

- Logs contain event messages and severity information.
- Metrics contain numerical measurements over time.
- Traces contain spans, relationships, and execution timing information.

Keeping appropriate table structures for each signal makes it easier to query and analyze the data.

### 22.4 Database and Table Verification

Database verification:

```sql
SHOW DATABASES;
```

Table verification:

```sql
SHOW TABLES FROM observability;
```

To inspect a table definition:

```sql
SHOW CREATE TABLE observability.<table_name>;
```

The database and required observability table creation were completed.

**Implementation Note:** The existence of tables confirms schema preparation. Actual application log, metric, and trace ingestion must be validated separately after their respective collectors are configured.

---

## 23. Demo Microservices Architecture

Two independent Python FastAPI applications were developed to simulate a real-world microservice communication flow.

The applications are:

1. Order API
2. Payment Service

### 23.1 Application Architecture

```text
                 Client
                   |
                   |
                   v
              Order API
                   |
                   |
            HTTP Request
                   |
                   v
            Payment Service
                   |
                   |
             Payment Result
                   |
                   v
              Order API
                   |
                   v
             Client Response
```

### 23.2 Order API Responsibilities

The Order API is responsible for:

- Accepting order creation requests.
- Generating unique order IDs.
- Storing order information in memory.
- Calling Payment Service through HTTP.
- Receiving payment responses.
- Generating structured JSON logs.
- Creating OpenTelemetry spans.
- Exposing application metrics.

### 23.3 Payment Service Responsibilities

The Payment Service is responsible for:

- Accepting payment requests.
- Generating unique payment IDs.
- Processing payment requests.
- Returning successful payment responses.
- Rejecting payments based on the configured amount condition.
- Generating structured JSON logs.
- Creating OpenTelemetry spans.
- Exposing application metrics.

### 23.4 Application Communication

The Order API communicates with Payment Service using HTTP.

The Kubernetes Service URL is:

```text
http://payment-service:8001/payments
```

The URL uses Kubernetes internal DNS resolution.

Both applications are deployed in the same Kubernetes namespace:

```text
applications
```

Therefore, the short Kubernetes Service name `payment-service` can be used for internal communication.

---

## 24. Order API Implementation

### 24.1 Project Location

```text
application/order-api/
```

Files:

```text
order-api/
├── main.py
├── requirements.txt
└── Dockerfile
```

### 24.2 Technologies Used

| Technology | Purpose |
|---|---|
| Python | Application development |
| FastAPI | REST API framework |
| Uvicorn | ASGI application server |
| HTTPX | HTTP communication with Payment Service |
| OpenTelemetry SDK | Distributed tracing |
| FastAPI Instrumentation | Automatic request instrumentation |
| HTTPX Instrumentation | HTTP client instrumentation and trace propagation |
| prometheus-client | Application metrics |

### 24.3 API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Application health check |
| POST | `/orders` | Create an order |
| GET | `/orders/{order_id}` | Retrieve order details |
| POST | `/simulate-error` | Generate a test application error |
| GET | `/metrics` | Expose Prometheus-compatible metrics |

### 24.4 Order Creation Workflow

When a client sends an order request:

1. Order API receives the customer ID and amount.
2. A unique order ID is generated.
3. Order information is stored in memory.
4. An INFO-level JSON log is generated.
5. Order API calls Payment Service.
6. Payment Service processes the payment.
7. Order API receives the payment response.
8. The final order response is returned to the client.

### 24.5 Payment Service URL Configuration

The Payment Service URL is configurable through an environment variable.

```python
SERVICE_NAME = "order-api"

PAYMENT_SERVICE_URL = os.getenv(
    "PAYMENT_SERVICE_URL",
    "http://payment-service:8001/payments"
)

OTLP_ENDPOINT = os.getenv(
    "OTEL_EXPORTER_OTLP_TRACES_ENDPOINT"
)
```

This allows the same application image to work in different environments by changing environment variables.

---

## 25. Payment Service Implementation

### 25.1 Project Location

```text
application/payment-service/
```

Files:

```text
payment-service/
├── main.py
├── requirements.txt
└── Dockerfile
```

### 25.2 Technologies Used

| Technology | Purpose |
|---|---|
| Python | Application development |
| FastAPI | REST API framework |
| Uvicorn | ASGI application server |
| OpenTelemetry SDK | Distributed tracing |
| FastAPI Instrumentation | Request instrumentation |
| prometheus-client | Application metrics |

### 25.3 API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Application health check |
| POST | `/payments` | Process payment |
| POST | `/simulate-error` | Generate a test application error |
| GET | `/metrics` | Expose Prometheus-compatible metrics |

### 25.4 Payment Processing Logic

The Payment Service uses a simple condition to simulate successful and failed payment processing.

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

Payment behavior:

| Amount | Expected Result |
|---|---|
| Less than 10000 | SUCCESS |
| Greater than or equal to 10000 | PAYMENT_DECLINED |

This condition is used to generate both successful and failed application workflows.

---

## 26. Python Dependencies and Virtual Environment

### 26.1 Order API Requirements

File:

```text
application/order-api/requirements.txt
```

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

### 26.2 Payment Service Requirements

File:

```text
application/payment-service/requirements.txt
```

```text
fastapi
uvicorn[standard]
opentelemetry-api
opentelemetry-sdk
opentelemetry-instrumentation-fastapi
opentelemetry-exporter-otlp
prometheus-client
```

### 26.3 Python Virtual Environment

A shared Python virtual environment was created inside the application directory.

```text
application/.venv/
```

The virtual environment was used to install dependencies and validate both applications locally.

Dependency verification:

```bash
.venv/bin/python -c \
"import fastapi, httpx, opentelemetry, prometheus_client; print('All required packages imported successfully')"
```

Verified output:

```text
All required packages imported successfully
```

This confirmed that the required Python packages were installed and importable.

---

## 27. Structured JSON Logging Implementation

Both applications were configured to generate structured JSON logs.

Instead of generating only plain-text log messages, each application produces JSON records containing useful application and tracing information.

### 27.1 Common Log Fields

| Field | Description |
|---|---|
| timestamp | UTC timestamp |
| level | Log severity |
| service_name | Application service name |
| message | Log message |
| trace_id | Distributed trace identifier |
| span_id | Current span identifier |

Additional fields depend on the application event.

For example:

- `order_id`
- `payment_id`
- `customer_id`
- `amount`
- `payment_status`
- `error_type`

### 27.2 Example Order API Log

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

### 27.3 Example Payment Service Error Log

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

### 27.4 Why Structured JSON Logging?

Structured JSON logging provides several advantages:

- Logs can be parsed programmatically.
- Fields can be extracted for filtering.
- Logs can be grouped by service name.
- Errors can be filtered using severity.
- Trace IDs can be used to correlate logs across services.
- JSON records can be ingested into ClickHouse for analytical queries.

---

## 28. OpenTelemetry Instrumentation

Both microservices were instrumented using the OpenTelemetry SDK.

### 28.1 OpenTelemetry Components

The applications use:

| Component | Purpose |
|---|---|
| OpenTelemetry API | Provides tracing interfaces |
| OpenTelemetry SDK | Creates and manages spans |
| TracerProvider | Configures tracing |
| Resource | Identifies the application service |
| BatchSpanProcessor | Processes spans in batches |
| OTLPSpanExporter | Exports spans through OTLP |
| FastAPI Instrumentation | Instruments incoming HTTP requests |
| HTTPX Instrumentation | Instruments outgoing HTTP requests |

### 28.2 Service Identification

Each application has a separate service name.

Order API:

```python
SERVICE_NAME = "order-api"
```

Payment Service:

```python
SERVICE_NAME = "payment-service"
```

The service name is attached to the OpenTelemetry resource.

### 28.3 Trace Context Propagation

The Order API uses HTTPX to communicate with Payment Service.

HTTPX instrumentation is enabled:

```python
HTTPXClientInstrumentor().instrument()
```

This instrumentation automatically propagates W3C trace context through HTTP headers.

The Payment Service receives the propagated context and creates its own span within the same trace.

### 28.4 Trace Relationship

```text
                  Trace ID
                     |
                     |
              Order API Span
                     |
                     |
               HTTP Request
                     |
                     v
             Payment Service Span
```

Both spans belong to the same distributed trace but have different span IDs.

### 28.5 OTLP Export Configuration

Both applications support an environment variable:

```text
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
```

This variable will be configured with the OpenTelemetry Collector OTLP HTTP endpoint in a subsequent phase.

**Current status:** The applications generate trace and span identifiers, and trace context propagation has been validated. Actual OTLP export to the Collector is not yet configured.

---

## 29. Application Metrics

Both applications expose Prometheus-compatible metrics through their `/metrics` endpoints.

### 29.1 Order API Metrics

| Metric | Type | Purpose |
|---|---|---|
| `orders_created_total` | Counter | Total orders created |
| `order_request_duration_seconds` | Histogram | Time taken to process order requests |

### 29.2 Payment Service Metrics

| Metric | Type | Purpose |
|---|---|---|
| `payments_processed_total` | Counter | Total payments processed, grouped by status |
| `payment_request_duration_seconds` | Histogram | Time taken to process payment requests |

### 29.3 Metrics Endpoints

Order API:

```text
http://localhost:8000/metrics
```

Payment Service:

```text
http://localhost:8001/metrics
```

The endpoints are implemented and available through the applications.

Scraping and exporting these metrics into ClickHouse through the OpenTelemetry Collector remains pending.

---

## 30. Docker Image Creation

Both applications were containerized using separate Dockerfiles.

### 30.1 Order API Dockerfile

Location:

```text
application/order-api/Dockerfile
```

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 30.2 Payment Service Dockerfile

Location:

```text
application/payment-service/Dockerfile
```

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8001

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
```

### 30.3 Container Port Configuration

| Application | Container Port |
|---|---:|
| Order API | 8000 |
| Payment Service | 8001 |

Separate container ports allow the applications to listen on different ports.

### 30.4 Build Order API Image

```bash
cd ~/clickhouse-unified-poc/application/order-api

docker build -t order-api:poc-v1 .
```

### 30.5 Build Payment Service Image

```bash
cd ~/clickhouse-unified-poc/application/payment-service

docker build -t payment-service:poc-v1 .
```

### 30.6 Verify Docker Images

```bash
docker images | grep -E 'order-api|payment-service'
```

Verified images:

| Image | Image Size |
|---|---:|
| order-api:poc-v1 | Approximately 307 MB |
| payment-service:poc-v1 | Approximately 305 MB |

Both Docker images were built successfully.

---

## 31. Loading Images into Kind

The locally built images were loaded into the Kind cluster.

This allows Kubernetes to use the local images without pulling them from Docker Hub.

### 31.1 Verify Active Kubernetes Context

```bash
kubectl config get-contexts
```

Active context:

```text
kind-clickhouse-unified-poc
```

### 31.2 Load Order API Image

```bash
kind load docker-image order-api:poc-v1 \
  --name clickhouse-unified-poc
```

### 31.3 Load Payment Service Image

```bash
kind load docker-image payment-service:poc-v1 \
  --name clickhouse-unified-poc
```

Both images were loaded into the target Kind cluster.

**Important:** Image loading is performed at the Kind cluster level. Kubernetes namespaces are specified separately in Deployment and Service manifests.

---

## 32. Kubernetes Application Deployment

### 32.1 Application Namespace

The `applications` namespace was already available in the cluster.

Verification:

```bash
kubectl get namespace applications
```

Verified output:

```text
NAME           STATUS
applications   Active
```

Initially, no application resources were present in the namespace.

```bash
kubectl get all -n applications
```

Output:

```text
No resources found in applications namespace.
```

Both microservices were subsequently deployed into this namespace.

### 32.2 Order API Kubernetes Manifest

File:

```text
kubernetes/order-api.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
  namespace: applications
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-api
  template:
    metadata:
      labels:
        app: order-api
    spec:
      containers:
        - name: order-api
          image: order-api:poc-v1
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8000
              protocol: TCP
          env:
            - name: PAYMENT_SERVICE_URL
              value: "http://payment-service:8001/payments"

---
apiVersion: v1
kind: Service
metadata:
  name: order-api
  namespace: applications
spec:
  type: ClusterIP
  selector:
    app: order-api
  ports:
    - name: http
      port: 8000
      targetPort: 8000
      protocol: TCP
```

### 32.3 Payment Service Kubernetes Manifest

File:

```text
kubernetes/payment-service.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: applications
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        - name: payment-service
          image: payment-service:poc-v1
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8001
              protocol: TCP

---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: applications
spec:
  type: ClusterIP
  selector:
    app: payment-service
  ports:
    - name: http
      port: 8001
      targetPort: 8001
      protocol: TCP
```

### 32.4 Kubernetes Manifest Validation

Before applying the manifests, a client-side dry run was performed.

```bash
kubectl apply --dry-run=client \
  -f ~/clickhouse-unified-poc/kubernetes/order-api.yaml \
  -f ~/clickhouse-unified-poc/kubernetes/payment-service.yaml
```

Verified output:

```text
deployment.apps/order-api created (dry run)
service/order-api created (dry run)
deployment.apps/payment-service created (dry run)
service/payment-service created (dry run)
```

All four resources passed dry-run validation.

### 32.5 Deploy Applications

```bash
kubectl apply \
  -f ~/clickhouse-unified-poc/kubernetes/order-api.yaml \
  -f ~/clickhouse-unified-poc/kubernetes/payment-service.yaml
```

Verified output:

```text
deployment.apps/order-api created
service/order-api created
deployment.apps/payment-service created
service/payment-service created
```

Both Deployments and Services were successfully created.

---

## 33. Kubernetes Pod and Service Validation

### 33.1 Pod Validation

Command:

```bash
kubectl get pods -n applications
```

Verified output:

```text
NAME                               READY   STATUS
order-api-64bfd6ccc7-qxfq5         1/1     Running
payment-service-7c86795b68-s4dhr   1/1     Running
```

Both application Pods reached the Running state.

### 33.2 ImagePullBackOff Issue

During the initial Payment Service deployment, the Pod entered the `ImagePullBackOff` state.

The Kubernetes events showed that the node attempted to pull the image from Docker Hub:

```text
docker.io/library/payment-service:poc-v1
```

The local image was not available to the Kind node at that time.

The issue was resolved by loading the local image into the cluster:

```bash
kind load docker-image payment-service:poc-v1 \
  --name clickhouse-unified-poc
```

After the image was available, the Payment Service Pod started successfully.

### 33.3 Service Validation

Command:

```bash
kubectl get svc -n applications
```

Verified output:

```text
NAME              TYPE        CLUSTER-IP      PORT(S)
order-api         ClusterIP   10.96.97.33     8000/TCP
payment-service   ClusterIP   10.96.108.140   8001/TCP
```

Service configuration:

| Service | Type | Port | Target Port |
|---|---|---:|---:|
| order-api | ClusterIP | 8000 | 8000 |
| payment-service | ClusterIP | 8001 | 8001 |

Both Services are available inside the Kubernetes cluster.

---

## 34. Internal Kubernetes Communication Validation

Internal communication between the two microservices was tested from inside the Order API Pod.

Command:

```bash
kubectl exec -it deployment/order-api \
  -n applications -- \
  python -c "import urllib.request; print(urllib.request.urlopen('http://payment-service:8001/health').read().decode())"
```

Verified response:

```json
{
  "status": "healthy",
  "service": "payment-service"
}
```

This confirms:

- Kubernetes Service DNS resolution is working.
- Order API can reach Payment Service.
- The Payment Service health endpoint is responding.
- Internal application communication is working.

---

## 35. Swagger UI and Browser Access

Both applications provide Swagger UI through FastAPI.

Kubernetes Services are configured as ClusterIP, so `kubectl port-forward` is used to access them locally.

### 35.1 Order API Port Forwarding

Run in Terminal 1:

```bash
kubectl port-forward \
  -n applications \
  svc/order-api 8000:8000
```

Browser URL:

```text
http://localhost:8000/docs
```

### 35.2 Payment Service Port Forwarding

Run in Terminal 2:

```bash
kubectl port-forward \
  -n applications \
  svc/payment-service 8001:8001
```

Browser URL:

```text
http://localhost:8001/docs
```

Both port-forward processes must remain running while accessing their respective Swagger UIs.

---

## 36. Application End-to-End Testing

Two application scenarios were validated through the Order API Swagger UI.

1. Successful payment.
2. Failed payment.

### 36.1 Successful Payment Scenario

Request:

```http
POST /orders
```

Request body:

```json
{
  "customer_id": "CUST-2026-201",
  "amount": 750
}
```

**Expected Workflow:**

1. Order API receives the request.
2. A unique order ID is generated.
3. Order API sends a payment request to Payment Service.
4. Payment Service processes the payment.
5. Payment Service returns a successful response.
6. Order API returns the order and payment details.

Observed response:

```text
HTTP 200 OK
```

The payment status returned was:

```text
SUCCESS
```

The successful application workflow was validated.

### 36.2 Failed Payment Scenario

Request:

```http
POST /orders
```

Request body:

```json
{
  "customer_id": "CUST-2026-202",
  "amount": 15000
}
```

**Expected Workflow:**

1. Order API receives the request.
2. An order ID is generated.
3. Order API sends the payment request to Payment Service.
4. Payment Service checks the requested amount.
5. Since the amount is greater than or equal to 10,000, the payment is rejected.
6. Payment Service returns HTTP 400.
7. Order API catches the HTTP error.
8. Order API generates an ERROR log.
9. Order API returns HTTP 502 to the client.

Observed Order API response:

```text
502 Bad Gateway
```

Response body:

```json
{
  "detail": "Payment Service request failed"
}
```

Observed Payment Service response:

```text
400 Bad Request
```

The failed payment scenario was validated successfully.

---

## 37. Application Log Verification

Application logs were inspected directly from Kubernetes using `kubectl logs`.

### 37.1 Payment Service Logs

Command:

```bash
kubectl logs deployment/payment-service \
  -n applications --tail=20
```

The logs showed successful payment processing:

```json
{
  "level": "INFO",
  "service_name": "payment-service",
  "message": "Payment processed successfully",
  "amount": 750.0
}
```

The failed payment scenario generated:

```json
{
  "level": "ERROR",
  "service_name": "payment-service",
  "message": "Payment processing failed",
  "amount": 15000.0,
  "error_type": "PAYMENT_DECLINED"
}
```

The application returned:

```text
POST /payments HTTP/1.1" 400 Bad Request
```

### 37.2 Order API Logs

Command:

```bash
kubectl logs deployment/order-api \
  -n applications --tail=20
```

The logs showed order creation:

```json
{
  "level": "INFO",
  "service_name": "order-api",
  "message": "Order created successfully",
  "amount": 15000.0
}
```

The failed payment scenario generated:

```json
{
  "level": "ERROR",
  "service_name": "order-api",
  "message": "Payment processing failed",
  "error": "Client error '400 Bad Request'"
}
```

The application returned:

```text
POST /orders HTTP/1.1" 502 Bad Gateway
```

**Validation Result:** Both applications successfully generated structured JSON logs for successful and failed application workflows.

---

## 38. Distributed Trace Context Propagation Validation

The two microservices were instrumented using OpenTelemetry.

The Order API generates a trace while processing an order.

When it calls Payment Service, HTTPX instrumentation propagates the W3C trace context through HTTP headers.

Payment Service receives this context and generates its own span within the same distributed trace.

### 38.1 Trace Correlation Architecture

```text
                Distributed Trace
                       |
                       |
                  Trace ID
                       |
             +---------+---------+
             |                   |
             v                   v
        Order API           Payment Service
             |                   |
             v                   v
         Span ID A            Span ID B
             |                   |
             +---------+---------+
                       |
                       v
                Same Trace ID
```

### 38.2 Failed Payment Trace Verification

For the failed payment scenario, both applications generated the following Trace ID:

```text
19f538ceb85f2ab83b81df829f2f6dca
```

Order API:

```text
Service Name: order-api
Trace ID: 19f538ceb85f2ab83b81df829f2f6dca
Span ID: e7746ae86dcf3a3f
```

Payment Service:

```text
Service Name: payment-service
Trace ID: 19f538ceb85f2ab83b81df829f2f6dca
Span ID: 0d93d73328f4d2a1
```

### 38.3 Trace Correlation Result

| Attribute | Order API | Payment Service |
|---|---|---|
| Trace ID | Same | Same |
| Span ID | e7746ae86dcf3a3f | 0d93d73328f4d2a1 |
| Service Name | order-api | payment-service |
| Log Level | ERROR | ERROR |
| Error Message | Payment processing failed | Payment processing failed |
| Error Type | HTTP 400 from Payment Service | PAYMENT_DECLINED |

### 38.4 Interpretation

The same Trace ID across both applications confirms that distributed trace context propagation is working.

Different Span IDs identify the individual operations performed by each service.

This provides the foundation for correlating application logs and distributed traces using:

- `trace_id`
- `span_id`
- `service_name`

**Important:** This validation confirms trace context propagation through application logs. It does not confirm that complete span records have been exported to ClickHouse.

The OTLP Collector endpoint has not yet been configured in the Kubernetes Deployments.

Actual trace ingestion remains pending.

---

## 39. Phase 3 Validation Summary

| Validation | Result |
|---|---|
| Observability database preparation | Completed |
| Observability table creation | Completed |
| Order API development | Completed |
| Payment Service development | Completed |
| Python dependency installation | Completed |
| Structured JSON logging | Completed |
| OpenTelemetry SDK instrumentation | Completed |
| HTTP trace context propagation | Successful |
| Application metrics endpoints | Implemented |
| Order API Docker image | Built |
| Payment Service Docker image | Built |
| Images loaded into Kind | Completed |
| Kubernetes manifest dry-run | Successful |
| Order API Deployment | Running |
| Payment Service Deployment | Running |
| Kubernetes Services | Available |
| Internal Service DNS communication | Successful |
| Order API Swagger UI | Verified |
| Payment Service Swagger UI | Verified |
| Successful payment test | HTTP 200 |
| Failed payment test | HTTP 502 from Order API, HTTP 400 from Payment Service |
| Application logs | Verified in Kubernetes |
| Same Trace ID across services | Verified |
| Fluent Bit log ingestion into ClickHouse | Pending |
| Metrics ingestion into ClickHouse | Pending |
| Trace ingestion into ClickHouse | Pending |

---

## 40. Updated Implementation Roadmap

| No. | Task | Status |
|---|---|---|
| 1 | Create project workspace and directory structure | Completed |
| 2 | Record system specifications | Completed |
| 3 | Install project-local kubectl | Completed |
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
| 16 | Create observability database | Completed |
| 17 | Create observability tables | Completed |
| 18 | Develop Order API | Completed |
| 19 | Develop Payment Service | Completed |
| 20 | Implement structured JSON logging | Completed |
| 21 | Instrument applications using OpenTelemetry SDK | Completed |
| 22 | Implement trace context propagation | Completed |
| 23 | Implement application metrics endpoints | Completed |
| 24 | Build Docker images | Completed |
| 25 | Load images into Kind | Completed |
| 26 | Deploy demo applications in Kubernetes | Completed |
| 27 | Validate application-to-application communication | Completed |
| 28 | Validate successful and failed application workflows | Completed |
| 29 | Verify application logs in Kubernetes | Completed |
| 30 | Verify trace context propagation | Completed |
| 31 | Configure Fluent Bit direct ClickHouse output | Pending |
| 32 | Deploy Fluent Bit DaemonSet | Pending |
| 33 | Validate application log ingestion into ClickHouse | Pending |
| 34 | Deploy Node Exporter, cAdvisor, and kube-state-metrics | Pending |
| 35 | Deploy OpenTelemetry Collector | Pending |
| 36 | Configure Prometheus Receiver | Pending |
| 37 | Validate metrics ingestion into ClickHouse | Pending |
| 38 | Configure OTLP Receiver | Pending |
| 39 | Configure application OTLP export | Pending |
| 40 | Validate trace ingestion into ClickHouse | Pending |
| 41 | Configure Grafana ClickHouse datasource | Pending |
| 42 | Create logs, metrics, and traces dashboards | Pending |
| 43 | Validate end-to-end observability | Pending |
| 44 | Validate cross-signal correlation | Pending |
| 45 | Document POC results and findings | Pending |

---

## 41. Current Implementation Status

At the completion of Phase 3:

- ClickHouse infrastructure is deployed and operational.
- The `observability` database and required tables have been created.
- Order API and Payment Service have been developed.
- Both applications have been containerized.
- Both application images have been loaded into the Kind cluster.
- Both applications are running in the Kubernetes `applications` namespace.
- Kubernetes Service communication is working.
- Swagger UI access is available through port forwarding.
- Successful and failed payment scenarios have been tested.
- Structured JSON logs are being generated.
- Distributed trace context propagation has been verified through matching Trace IDs.

### Remaining Work

The next implementation phase focuses on connecting the application logs to ClickHouse.

The planned work is:

1. Verify the actual ClickHouse log table schema.
2. Review the Fluent Bit configuration.
3. Deploy Fluent Bit as a DaemonSet.
4. Collect Kubernetes container logs.
5. Send application logs directly to ClickHouse.
6. Generate new successful and failed application requests.
7. Validate the ingested logs using ClickHouse SQL queries.

After log ingestion, the POC will proceed with infrastructure metrics and distributed trace ingestion through the OpenTelemetry Collector.

---

## 42. Important POC Notes and Limitations

- This is a local functional POC using Kind.
- The cluster contains one control-plane node.
- ClickHouse is deployed with one shard and one replica.
- Keeper is deployed with one replica.
- Separate Persistent Volume Claims are configured for ClickHouse and Keeper.
- Demo application data is stored in memory; no external application database is used.
- Both applications run in separate Kubernetes Deployments.
- Kubernetes Services are exposed internally through ClusterIP.
- Browser access is provided through `kubectl port-forward`.
- Fluent Bit will handle logs independently and send them directly to ClickHouse.
- OpenTelemetry Collector will handle metrics and traces.
- Application logs have been verified in Kubernetes stdout, not yet in ClickHouse.
- Trace context propagation has been verified, but complete OTLP span export is pending.
- Application metrics endpoints are implemented, but metrics scraping and export are pending.
- This POC does not represent a production-grade HA deployment.

## 43. Next Implementation Phase

The next phase will focus on **Fluent Bit integration and application log ingestion into ClickHouse**.

The implementation will follow this flow:

```text
Order API                 Payment Service
    |                            |
    |                            |
    +------------+---------------+
                 |
                 v
        Kubernetes Container
             stdout
                 |
                 v
          Fluent Bit
          DaemonSet
                 |
                 v
             ClickHouse
                 |
                 v
           SQL Validation
```

The primary validation goal will be to confirm that logs generated by both applications are successfully collected by Fluent Bit and stored in the correct ClickHouse table.

The validation will include:

- INFO log ingestion.
- ERROR log ingestion.
- Service name extraction.
- Timestamp validation.
- Trace ID and Span ID preservation.
- Querying logs by service name.
- Querying logs by severity.
- Correlating logs using Trace ID.

**Current Milestone:** ClickHouse infrastructure, observability schema preparation, and demo microservice deployment have been completed. Both application workflows and distributed trace context propagation have been validated. The next milestone is to establish and validate the Fluent Bit-to-ClickHouse logs pipeline.

# PHASE 4: Fluent Bit Integration and ClickHouse Log Ingestion

## 44. Phase 4 Overview

The objective of Phase 4 was to implement the Kubernetes log collection pipeline using Fluent Bit and send container logs directly to ClickHouse.

This phase focused on:

1. Configuring Fluent Bit to collect Kubernetes container logs.
2. Enriching logs with Kubernetes metadata.
3. Transforming log records using a custom Lua script.
4. Sending transformed logs directly to ClickHouse.
5. Validating log ingestion for infrastructure and application workloads.
6. Verifying structured application logs and error logs.
7. Validating trace ID and span ID correlation across microservices.

At the end of this phase, Fluent Bit was successfully deployed as a DaemonSet, and Kubernetes container logs were being ingested into ClickHouse.

## 45. Logging Pipeline Architecture

The following architecture was implemented for log ingestion:

```text
        Kubernetes Workloads
                 |
                 v
        Container stdout/stderr
                 |
                 v
          Fluent Bit Agent
             (DaemonSet)
                 |
                 v
          Tail Input Plugin
                 |
                 v
       Kubernetes Filter Plugin
                 |
                 v
          Lua Transformation
                 |
                 v
          HTTP Output Plugin
                 |
                 v
             ClickHouse
                 |
                 v
 observability.kubernetes_logs
```

### Workflow

- Kubernetes workloads generate container logs through stdout and stderr.
- Fluent Bit collects container log files from Kubernetes nodes.
- The Kubernetes filter enriches logs with namespace, pod, and container metadata.
- The Lua script transforms log records into the required ClickHouse table structure.
- The HTTP output plugin sends transformed records directly to ClickHouse.
- ClickHouse stores the logs in the `observability.kubernetes_logs` table.

**Important:** Fluent Bit sends logs directly to ClickHouse. The OpenTelemetry Collector is not involved in this logging pipeline. It will be used separately for metrics and traces.

## 46. ClickHouse Log Table Creation

A dedicated table named `kubernetes_logs` was created inside the `observability` database.

### Table Schema

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

### Column Description

| Column | Description |
|---|---|
| timestamp | Timestamp assigned to the log event |
| cluster_name | Kubernetes cluster name |
| namespace | Kubernetes namespace |
| pod_name | Name of the Kubernetes pod |
| container_name | Container name |
| level | Log severity level |
| service_name | Application or service name |
| message | Extracted log message |
| trace_id | Distributed trace identifier |
| span_id | Span identifier |
| attributes | Additional structured attributes |
| log_type | Log category such as application, access, or infrastructure |
| raw_log | Original log record |

### Table Design Considerations

- `MergeTree` is used as the storage engine.
- Monthly partitioning is configured using the log timestamp.
- The sorting key uses namespace, pod name, and timestamp.
- `LowCardinality(String)` is used for frequently repeated string values such as namespace, container name, and log level.
- The schema supports both structured application logs and plain-text infrastructure logs.

## 47. Fluent Bit Installation and Helm Configuration

Fluent Bit was deployed using its official Helm chart.

### Add Fluent Bit Helm Repository

```bash
helm repo add fluent https://fluent.github.io/helm-charts
```

Update the repository:

```bash
helm repo update
```

### Download Helm Chart

The Fluent Bit chart version used for this POC was `0.58.3`.

```bash
helm pull fluent/fluent-bit --version 0.58.3 \
  --destination ~/clickhouse-unified-poc/helm
```

### Configuration Files

The following files were maintained:

```text
clickhouse-unified-poc/
├── configs/
│   ├── fluent-bit-default-values.yaml
│   └── fluent-bit-values.yaml
└── helm/
    └── fluent-bit-0.58.3.tgz
```

- `fluent-bit-default-values.yaml` contains the chart's default configuration.
- `fluent-bit-values.yaml` contains the custom POC configuration.

### Configuration Validation

Helm lint was executed to validate the custom configuration.

```bash
helm lint ~/clickhouse-unified-poc/helm/fluent-bit-0.58.3.tgz \
  -f ~/clickhouse-unified-poc/configs/fluent-bit-values.yaml
```

The Helm templates were also rendered and inspected before deployment.

## 48. Fluent Bit Input Configuration

Fluent Bit uses the Tail input plugin to collect Kubernetes container logs.

```ini
[INPUT]
    Name tail
    Path /var/log/containers/*.log
    multiline.parser docker, cri
    Tag kube.*
    Mem_Buf_Limit 20MB
    Skip_Long_Lines On
```

### Configuration Explanation

- `Name tail`: Uses the Tail input plugin.
- `Path`: Reads container log files from `/var/log/containers/`.
- `multiline.parser`: Supports Docker and CRI container log formats.
- `Tag`: Assigns the `kube.*` tag to collected records.
- `Mem_Buf_Limit`: Limits the memory buffer used by the input.
- `Skip_Long_Lines`: Skips log lines that exceed the configured buffer constraints.

The Tail input plugin is responsible for collecting log records. It does not perform Kubernetes metadata enrichment or ClickHouse transformation.

## 49. Kubernetes Filter Configuration

The Kubernetes filter enriches collected logs with Kubernetes metadata.

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

### Configuration Explanation

- `Name kubernetes`: Enables the Kubernetes filter.
- `Match kube.*`: Applies the filter to records with matching tags.
- `Kube_Tag_Prefix`: Defines the prefix used to identify Kubernetes container log records.
- `Merge_Log On`: Attempts to merge structured JSON log content into the main record.
- `Keep_Log On`: Retains the original log field after merging.
- `K8S-Logging.Parser On`: Allows pod-level parser configuration.
- `K8S-Logging.Exclude On`: Allows pod-level log exclusion configuration.

This filter helps attach Kubernetes metadata such as namespace, pod name, and container name to each log record.

## 50. Lua Transformation and Its Purpose

A custom Lua script was implemented to transform Fluent Bit records into the required ClickHouse table structure.

Script location:

```text
configs/clickhouse_transform.lua
```

Inside the Fluent Bit container, the script is mounted at:

```text
/fluent-bit/scripts/clickhouse_transform.lua
```

### Why Lua Was Used

Fluent Bit provides several built-in filters, including:

- Kubernetes Filter
- Parser Filter
- Modify Filter
- Nest Filter

These filters support common parsing, enrichment, and field modification operations.

However, this POC requires additional conditional processing, including:

- Extracting fields from structured log records.
- Applying default values when fields are missing.
- Normalizing log severity to uppercase.
- Classifying records into application, access, and infrastructure categories.
- Formatting the event timestamp.
- Producing a consistent output structure matching the ClickHouse table.

A Lua filter was selected to implement this custom transformation logic in one place.

**Lua is not mandatory for every Fluent Bit deployment.** It is used here because the POC requires custom conditional transformation beyond the standard filter configuration.

### Lua Script

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

### Lua Script Working

1. Reads Kubernetes metadata from the record.
2. Extracts namespace, pod name, and container name.
3. Reads the original log content.
4. Extracts level, service name, message, trace ID, and span ID when available.
5. Converts the log level to uppercase.
6. Classifies the record:
   - `application` when `service_name` is present.
   - `access` when the basic HTTP access-log pattern matches.
   - `infrastructure` otherwise.
7. Formats the Fluent Bit event timestamp into a UTC timestamp with millisecond precision.
8. Creates a transformed record containing the 13 ClickHouse table fields.
9. Returns the transformed record to Fluent Bit.

### Important Timestamp Detail

The timestamp used in the ClickHouse table is generated from the Fluent Bit event timestamp.

The original application timestamp, if present in the JSON log, remains available inside `raw_log`. It is not currently used as the normalized ClickHouse `timestamp`.

### Important Transformation Limitations

- The `attributes` column currently contains `{}`.
- Additional application fields such as `order_id`, `payment_id`, `customer_id`, `amount`, and `error_type` are not individually extracted into `attributes`.
- These fields remain available in the original `raw_log`.
- The current `log_type` classification is basic and can be enhanced for more log formats.
- Plain-text records may receive `UNKNOWN` as their log level when no severity is extracted.

### Lua Return Code

The script returns:

```lua
return 2, timestamp, output
```

Return code `2` tells Fluent Bit to replace the original record with the transformed record while retaining the event timestamp.

## 51. Fluent Bit Lua Filter Configuration

The Lua script is connected to Fluent Bit using the Lua filter.

```ini
[FILTER]
    Name lua
    Match kube.*
    script /fluent-bit/scripts/clickhouse_transform.lua
    call transform_log
```

### Configuration Explanation

- `Name lua`: Enables the Lua filter.
- `Match kube.*`: Applies the transformation to Kubernetes log records.
- `script`: Specifies the mounted Lua script path.
- `call`: Specifies the Lua function that Fluent Bit should execute.

The Lua filter processes the records after the Kubernetes filter has enriched them.

## 52. Fluent Bit HTTP Output Configuration

Fluent Bit sends transformed records directly to ClickHouse through its HTTP interface.

```ini
[OUTPUT]
    Name http
    Match kube.*
    Host clickhouse-clickhouse-headless.clickhouse.svc.cluster.local
    Port 8123
    URI /?query=INSERT%20INTO%20observability.kubernetes_logs%20FORMAT%20JSONEachRow
    Format json_lines
```

### Configuration Explanation

- `Name http`: Uses the HTTP output plugin.
- `Match kube.*`: Sends matching Kubernetes log records.
- `Host`: ClickHouse headless service DNS.
- `Port`: ClickHouse HTTP port 8123.
- `URI`: Executes an INSERT query against `observability.kubernetes_logs`.
- `Format json_lines`: Sends records in JSONEachRow-compatible line format.

The Fluent Bit output sends data to ClickHouse without routing it through the OpenTelemetry Collector.

## 53. Fluent Bit Deployment in Kubernetes

Fluent Bit was deployed in the `logging` namespace as a DaemonSet.

### Deployment Command

```bash
helm upgrade --install fluent-bit \
  ~/clickhouse-unified-poc/helm/fluent-bit-0.58.3.tgz \
  --namespace logging \
  -f ~/clickhouse-unified-poc/configs/fluent-bit-values.yaml \
  --wait --timeout 5m
```

### Deployment Validation

Check the Fluent Bit Pod:

```bash
kubectl get pods -n logging
```

Check the DaemonSet:

```bash
kubectl get daemonset -n logging
```

The deployment was validated with:

- Fluent Bit Pod in Running state.
- DaemonSet desired, current, ready, and available counts matching the single-node cluster.
- Fluent Bit logs showing successful HTTP responses from ClickHouse.

### ClickHouse Connectivity Validation

Connectivity from the logging namespace to ClickHouse was tested using its HTTP endpoint.

The `/ping` endpoint returned:

```text
HTTP 200
Ok.
```

This confirmed that Fluent Bit's namespace could reach the ClickHouse HTTP service.

## 54. Log Ingestion Validation in ClickHouse

After deploying Fluent Bit, log ingestion was validated using SQL queries.

### Verify Total Log Count

```sql
SELECT count()
FROM observability.kubernetes_logs;
```

The initial validation returned 619 records. The count increased as Fluent Bit continued collecting logs.

### Verify Logs by Namespace

```sql
SELECT
    namespace,
    count() AS total_logs
FROM observability.kubernetes_logs
GROUP BY namespace
ORDER BY total_logs DESC;
```

During validation, logs were observed from namespaces including:

- `clickhouse-operator-system`
- `logging`
- `kube-system`
- `clickhouse`

The counts are dynamic because Fluent Bit continuously collects new container logs.

### Verify Recent Logs

```sql
SELECT
    timestamp,
    namespace,
    pod_name,
    container_name,
    level,
    service_name,
    message,
    log_type
FROM observability.kubernetes_logs
ORDER BY timestamp DESC
LIMIT 20;
```

This query was used to inspect recent records and validate the transformed table structure.

## 55. Application Log Ingestion Validation

The Order API and Payment Service were tested to generate application logs.

A successful order request was triggered through the Order API.

The application logs were subsequently observed in ClickHouse.

### Verify Application Logs

```sql
SELECT
    timestamp,
    namespace,
    pod_name,
    service_name,
    level,
    message,
    trace_id,
    span_id,
    raw_log
FROM observability.kubernetes_logs
WHERE namespace = 'applications'
ORDER BY timestamp DESC
LIMIT 20;
```

The validation confirmed that:

- Order API logs were ingested.
- Payment Service logs were ingested.
- Structured log fields such as service name and severity were extracted.
- Trace IDs and span IDs were available in the corresponding application log records.
- Original log content was preserved in `raw_log`.

## 56. Successful and Failed Payment Log Validation

Both successful and failed payment scenarios were tested to validate application log ingestion.

### Successful Payment Scenario

A successful order request was generated using a payment amount below the configured rejection threshold.

Observed behavior:

- Order API processed the request.
- Payment Service processed the payment successfully.
- Order API returned HTTP 200.
- Application logs were available in ClickHouse.

### Failed Payment Scenario

A failed payment request was generated using an amount of 15000.

Observed behavior:

- Payment Service rejected the payment with HTTP 400.
- The error type was `PAYMENT_DECLINED`.
- Order API received the
