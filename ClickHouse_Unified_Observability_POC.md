# ClickHouse Unified Observability POC

## 1. Project Overview

The objective of this Proof of Concept (POC) is to evaluate **ClickHouse as a centralized observability backend** for storing and querying three major observability signals:

- Logs
- Metrics
- Traces

The POC uses a local Kubernetes environment powered by Kind. Three independent ingestion pipelines collect logs, metrics, and traces and send them to ClickHouse for centralized storage and analysis.

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

The following activities are included in this POC.

**1. Kubernetes Environment**
- Create a separate, single-node Kind cluster.
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
           +------+------+
           |      |      |
          Logs  Metrics Traces
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
| Log Collection | Fluent Bit | Kubernetes container log collection |
| Metrics Exporter | Node Exporter | Node-level metrics |
| Metrics Exporter | cAdvisor | Container-level metrics |
| Metrics Exporter | kube-state-metrics | Kubernetes object and state metrics |
| Metrics Scraping | OTel Collector Prometheus Receiver | Scrape Prometheus-compatible metrics |
| Application Instrumentation | OpenTelemetry SDK | Generate distributed traces |
| Trace Transport | OTLP | Send traces to Collector |
| Telemetry Collector | OpenTelemetry Collector | Collect metrics and traces |
| Storage Backend | ClickHouse | Centralized observability storage |
| Visualization | Grafana | Dashboards and visualization |

## 6. System Specifications

The POC is implemented in the following local environment.

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
| Kubernetes Node Image | v1.37.0 |

## 7. Project Setup and Directory Structure

### 7.1 Project Workspace

The project workspace was created at:

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
│   ├── kind-config.yaml
│   └── namespaces.yaml
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

**Verified Output:**

```text
Client Version: v1.37.1
Kustomize Version: v5.8.1
```

**Checksum Verification:**

```text
kubectl: OK
```

The project-local kubectl installation was completed successfully without changing the system-level kubectl version.

## 9. Kind Cluster Setup

A separate single-node Kind cluster was created specifically for this POC.

### 9.1 Cluster Details

| Component | Configuration |
|---|---|
| Cluster Name | clickhouse-unified-poc |
| Kubernetes Version | v1.37.0 |
| Node Configuration | 1 Control Plane |
| Node Image | kindest/node:v1.37.0 |
| Cluster Type | Local Kubernetes Cluster |

### 9.2 Kind Configuration File

File location:

```text
kubernetes/kind-config.yaml
```

Configuration:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
```

### 9.3 Cluster Creation

```bash
kind create cluster \
  --name clickhouse-unified-poc \
  --config kubernetes/kind-config.yaml
```

The cluster was created successfully.

### 9.4 Cluster Validation

Cluster information:

```bash
kubectl cluster-info \
  --context kind-clickhouse-unified-poc
```

Node validation:

```bash
kubectl --context kind-clickhouse-unified-poc get nodes -o wide
```

Pod validation:

```bash
kubectl --context kind-clickhouse-unified-poc get pods -A
```

**Validation Results:**

- Kubernetes API Server was accessible.
- The control-plane node reached `Ready` status.
- Kubernetes system pods were running.
- CoreDNS was running.
- Kind networking components were running.
- Local Path Storage provisioner was running.

### 9.5 StorageClass Validation

Command:

```bash
kubectl --context kind-clickhouse-unified-poc get storageclass
```

Verified StorageClass:

| Property | Value |
|---|---|
| StorageClass Name | standard |
| Provisioner | rancher.io/local-path |
| Reclaim Policy | Delete |
| Volume Binding Mode | WaitForFirstConsumer |
| Default StorageClass | Yes |
| Volume Expansion | No |

The default StorageClass is available for persistent volume provisioning.

## 10. Kubernetes Namespace Setup

Six separate namespaces were created to organize the POC components.

### 10.1 Namespace Configuration

File location:

```text
kubernetes/namespaces.yaml
```

Configuration:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: clickhouse
---
apiVersion: v1
kind: Namespace
metadata:
  name: logging
---
apiVersion: v1
kind: Namespace
metadata:
  name: metrics
---
apiVersion: v1
kind: Namespace
metadata:
  name: opentelemetry
---
apiVersion: v1
kind: Namespace
metadata:
  name: applications
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

### 10.2 Apply Namespaces

```bash
kubectl apply -f kubernetes/namespaces.yaml
```

### 10.3 Namespace Purpose

| Namespace | Purpose |
|---|---|
| clickhouse | ClickHouse database and related components |
| logging | Fluent Bit and logging components |
| metrics | Metrics exporters |
| opentelemetry | OpenTelemetry Collector |
| applications | Demo applications |
| monitoring | Grafana and monitoring components |

### 10.4 Namespace Validation

Command:

```bash
kubectl get namespaces
```

**Validation Result:**

All six namespaces were successfully created and their status was verified as `Active`.

## 11. Implementation Roadmap

| No. | Task | Status |
|---|---|---|
| 1 | Create project workspace and directory structure | Completed |
| 2 | Record system specifications | Completed |
| 3 | Install project-local kubectl v1.37.1 | Completed |
| 4 | Prepare Kind cluster configuration | Completed |
| 5 | Create a separate single-node Kind cluster | Completed |
| 6 | Validate Kubernetes node and system pods | Completed |
| 7 | Validate default StorageClass | Completed |
| 8 | Create Kubernetes namespaces | Completed |
| 9 | Deploy ClickHouse Operator | Pending |
| 10 | Deploy ClickHouse | Pending |
| 11 | Configure Fluent Bit direct ClickHouse output | Pending |
| 12 | Validate log ingestion | Pending |
| 13 | Deploy Node Exporter, cAdvisor, and kube-state-metrics | Pending |
| 14 | Configure OTel Collector Prometheus Receiver | Pending |
| 15 | Validate metrics ingestion | Pending |
| 16 | Develop Order API and Payment Service | Pending |
| 17 | Instrument applications using OpenTelemetry SDK | Pending |
| 18 | Configure OTel Collector OTLP Receiver | Pending |
| 19 | Validate trace ingestion | Pending |
| 20 | Deploy demo applications | Pending |
| 21 | Install Grafana | Pending |
| 22 | Configure ClickHouse datasource in Grafana | Pending |
| 23 | Create observability dashboards | Pending |
| 24 | Validate logs, metrics, and traces | Pending |
| 25 | Validate cross-signal correlation | Pending |
| 26 | Document POC results and findings | Pending |

## 12. Validation Strategy

The following validations will be performed during implementation.

### Logs

- Verify Fluent Bit collects Kubernetes container logs.
- Verify logs reach ClickHouse directly.
- Check stored log records using SQL queries.

### Metrics

- Verify exporters expose metrics.
- Verify OTel Collector successfully scrapes the endpoints.
- Verify metrics are exported and stored in ClickHouse.
- Query metrics from ClickHouse.

### Traces

- Verify demo applications generate traces.
- Verify traces reach the OTel Collector through OTLP.
- Verify traces are exported and stored in ClickHouse.
- Query trace records from ClickHouse.

### Visualization

- Verify Grafana connects to ClickHouse.
- Verify dashboards display logs, metrics, and traces.
- Validate the available correlation fields between logs and traces.

## 13. Current Project Status

### Completed

- Project workspace created.
- Required project directories created.
- System specifications documented.
- Project-local kubectl v1.37.1 downloaded.
- SHA256 checksum verified.
- Project-local kubectl installed and version verified.
- POC objectives, scope, and proposed architecture defined.
- Kind cluster configuration prepared.
- Separate single-node Kind cluster created successfully.
- Kubernetes control-plane node verified as Ready.
- Kubernetes system pods verified.
- Default StorageClass verified.
- Six Kubernetes namespaces created successfully.
- All six namespaces verified as Active.

### Pending

- ClickHouse Operator installation.
- ClickHouse deployment and database configuration.
- Fluent Bit log ingestion pipeline.
- Metrics exporters and OTel Collector configuration.
- Demo microservices and distributed tracing pipeline.
- Grafana deployment and dashboard creation.
- End-to-end validation.
- Cross-signal correlation testing.
- Final POC documentation and findings.

## 14. Expected Final Outcome

The final outcome will be a working local Kubernetes-based unified observability POC in which:

- Fluent Bit sends Kubernetes logs directly to ClickHouse.
- OTel Collector scrapes metrics from Prometheus-compatible exporters and sends them to ClickHouse.
- OpenTelemetry-instrumented applications send traces through OTLP to the Collector, which exports them to ClickHouse.
- Grafana visualizes all three observability signals.
- Logs and traces can be correlated using trace identifiers.

This POC will help evaluate ClickHouse's capabilities as a centralized backend for unified observability.
