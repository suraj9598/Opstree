# ClickHouse Unified Observability POC

## 1. Project Overview

The objective of this Proof of Concept (POC) is to evaluate **ClickHouse as a centralized observability backend** for storing and querying three major observability signals:

- Logs
- Metrics
- Traces

The POC will use a local Kubernetes environment powered by Kind. Three independent ingestion pipelines will collect logs, metrics, and traces and send them to ClickHouse for centralized storage and analysis.

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

The POC is planned for the following local environment.

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

## 9. Implementation Roadmap

The following activities are planned for the POC.

| No. | Task | Status |
|---|---|---|
| 1 | Create project workspace and directory structure | Completed |
| 2 | Record system specifications | Completed |
| 3 | Install project-local kubectl v1.37.1 | Completed |
| 4 | Prepare Kind cluster configuration | Pending |
| 5 | Create a separate single-node Kind cluster | Pending |
| 6 | Create Kubernetes namespaces | Pending |
| 7 | Deploy ClickHouse | Pending |
| 8 | Configure Fluent Bit direct ClickHouse output | Pending |
| 9 | Validate log ingestion | Pending |
| 10 | Deploy Node Exporter, cAdvisor, and kube-state-metrics | Pending |
| 11 | Configure OTel Collector Prometheus Receiver | Pending |
| 12 | Validate metrics ingestion | Pending |
| 13 | Develop Order API and Payment Service | Pending |
| 14 | Instrument applications using OpenTelemetry SDK | Pending |
| 15 | Configure OTel Collector OTLP Receiver | Pending |
| 16 | Validate trace ingestion | Pending |
| 17 | Deploy demo applications | Pending |
| 18 | Install Grafana | Pending |
| 19 | Configure ClickHouse datasource in Grafana | Pending |
| 20 | Create observability dashboards | Pending |
| 21 | Validate logs, metrics, and traces | Pending |
| 22 | Validate cross-signal correlation | Pending |
| 23 | Document POC results and findings | Pending |

## 10. Validation Strategy

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

## 11. Current Project Status

### Completed
- Project workspace created.
- Required project directories created.
- System specifications documented.
- Project-local kubectl v1.37.1 downloaded.
- SHA256 checksum verified.
- Project-local kubectl installed and version verified.
- POC objectives, scope, and proposed architecture defined.

### Pending
- Kind cluster configuration and creation.
- ClickHouse deployment.
- Fluent Bit log ingestion pipeline.
- Metrics exporters and OTel Collector configuration.
- Demo microservices and distributed tracing pipeline.
- Grafana deployment and dashboard creation.
- End-to-end validation.

## 12. Expected Final Outcome

The final outcome will be a working local Kubernetes-based unified observability POC in which:

- Fluent Bit sends Kubernetes logs directly to ClickHouse.
- OTel Collector scrapes metrics from Prometheus-compatible exporters and sends them to ClickHouse.
- OpenTelemetry-instrumented applications send traces through OTLP to the Collector, which exports them to ClickHouse.
- Grafana visualizes all three observability signals.
- Logs and traces can be correlated using trace identifiers.

This POC will help evaluate ClickHouse's capabilities as a centralized backend for unified observability.
