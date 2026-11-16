# High Availability Observability Stack on Kubernetes

## 1. Project Overview

The objective of this project is to deploy a High Availability (HA) Observability Stack on a local Kubernetes environment using Kind.

The observability stack will be used to collect, store, monitor, and visualize metrics, logs, and traces from Kubernetes workloads.

The setup will be implemented step by step using Kubernetes and Helm.

## 2. Project Objectives

- Set up a local Kubernetes cluster using Kind.
- Deploy observability components.
- Collect Kubernetes metrics, logs, and traces.
- Configure Grafana for visualization.
- Implement High Availability for supported components.
- Understand the architecture and communication between components.

## 3. Environment

| Component | Configuration |
|---|---|
| Kubernetes | Kind |
| Cluster Name | ha-observability |
| Control Plane Nodes | 1 |
| Worker Nodes | 2 |
| Total Nodes | 3 |
| Deployment Environment | Local |
| Package Manager | Helm |
| Container Runtime | Docker |

## 4. Kubernetes Cluster Setup

### Step 1: Create Project Directory

```bash
mkdir -p ~/ha-observability-poc
cd ~/ha-observability-poc
mkdir kind
```

### Step 2: Create Kind Configuration

Create the configuration file:

```bash
nano kind/kind-config.yaml
```

Configuration:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ha-observability

nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

### Step 3: Create the Kubernetes Cluster

```bash
kind create cluster --config kind/kind-config.yaml
```

This command creates a Kubernetes cluster with one control-plane node and two worker nodes.

### Step 4: Verify Cluster Nodes

```bash
kubectl get nodes
```

The expected output should show all three nodes in `Ready` status.
<img width="955" height="118" alt="image" src="https://github.com/user-attachments/assets/7d4af6ac-6049-49f4-97a8-29074eaa7f63" />

### Step 5: Verify Kubernetes Pods

```bash
kubectl get pods -A
```

This command verifies that the Kubernetes system pods are running.

## 5. Cluster Architecture

The initial Kubernetes cluster consists of:

- One control-plane node responsible for managing the cluster.
- Two worker nodes responsible for running application and observability workloads.

```text
              Kind Kubernetes Cluster
              ha-observability
                      |
          +-----------+-----------+
          |                       |
    Control Plane             Worker Nodes
          |                       |
    Cluster Management     +------+------+
                           |             |
                        Worker 1      Worker 2
                           |             |
                      Workloads     Workloads
```

## 6. Namespace Organization

To keep the observability stack organized, separate Kubernetes namespaces are created for each observability component group.

| Namespace  | Components                     | Purpose                        |
| ---------- | ------------------------------ | ------------------------------ |
| logging    | Loki, Fluent Bit               | Log collection and storage     |
| metrics    | VictoriaMetrics, vmagent       | Metrics collection and storage |
| tracing    | Tempo, OpenTelemetry Collector | Distributed tracing            |
| monitoring | Grafana                        | Visualization and dashboards   |

### Create Namespaces

```bash
kubectl create namespace logging
kubectl create namespace metrics
kubectl create namespace tracing
kubectl create namespace monitoring
```

### Verify Namespaces

```bash
kubectl get namespaces
```
<img width="948" height="225" alt="image" src="https://github.com/user-attachments/assets/e6bc41fd-3693-4518-8300-331c1a067bec" />

### Benefits

- Better organization of observability components.
- Easier resource management.
- Simplified troubleshooting.
- Namespace-level access control can be configured later.
- Cleaner separation between logs, metrics, traces, and visualization.

## 7. Observability Tools and Installation Plan

### 7.1 Overview

The High Availability Observability Stack consists of tools for collecting, storing, processing, and visualizing Kubernetes metrics, logs, and traces.

The stack is divided into three main observability areas:

- **Metrics Monitoring:** Monitors CPU, memory, disk, network, and Kubernetes resource health.
- **Log Monitoring:** Collects and stores application and Kubernetes logs.
- **Distributed Tracing:** Tracks application requests across multiple services.

Grafana is used as the central visualization platform.

### 7.2 Tools and Their Purpose

| Tool | Namespace | Category | Purpose |
|---|---|---|---|
| Node Exporter | metrics | Metrics | Collects node-level metrics such as CPU, memory, disk, and network usage. |
| kube-state-metrics | metrics | Metrics | Exposes Kubernetes object metrics, including Pod, Deployment, and Node status. |
| VictoriaMetrics | metrics | Metrics Storage | Stores metrics collected from Kubernetes and applications. |
| vmagent | metrics | Metrics Collection | Scrapes metrics from exporters and forwards them to VictoriaMetrics. |
| Loki | logging | Log Storage | Stores and indexes Kubernetes and application logs. |
| Fluent Bit | logging | Log Collection | Collects container logs from Kubernetes nodes and forwards them to Loki. |
| Tempo | tracing | Trace Storage | Stores distributed traces for application request analysis. |
| OpenTelemetry Collector | tracing | Telemetry Collection | Receives, processes, and exports telemetry data such as traces. |
| Grafana | monitoring | Visualization | Provides dashboards for metrics, logs, and traces. |

### 7.3 Namespace Organization

Each observability component group is deployed into a separate Kubernetes namespace.

| Namespace | Components |
|---|---|
| logging | Loki, Fluent Bit |
| metrics | Node Exporter, kube-state-metrics, VictoriaMetrics, vmagent |
| tracing | Tempo, OpenTelemetry Collector |
| monitoring | Grafana |

This namespace-based organization helps maintain a clean and manageable Kubernetes environment.

### 7.4 Installation Order

The observability stack will be installed in the following order:

| Phase | Tool |
|---|---|
| 1 | Node Exporter |
| 2 | kube-state-metrics |
| 3 | VictoriaMetrics |
| 4 | vmagent |
| 5 | Loki |
| 6 | Fluent Bit |
| 7 | Tempo |
| 8 | OpenTelemetry Collector |
| 9 | Grafana |

### 7.5 Architecture Overview

```text
                 Kubernetes Cluster
                        |
        +---------------+---------------+
        |               |               |
      Metrics          Logs           Traces
        |               |               |
 Node Exporter      Fluent Bit    OpenTelemetry
 kube-state-metrics     |           Collector
        |               |               |
     vmagent           Loki            Tempo
        |               |               |
 VictoriaMetrics        |               |
        +---------------+---------------+
                        |
                     Grafana
                  Visualization
```

### 7.6 Deployment Approach

- Kubernetes cluster is created using Kind.
- Each component is deployed into its designated namespace.
- Helm is used for installing and managing components wherever applicable.
- Each component is configured and verified before proceeding to the next component.
- README.md is updated incrementally with installation commands, configurations, and verification steps.
- The setup is designed to evaluate High Availability in a local Kubernetes environment.

### 7.7 Scope

The current project focuses on deploying a local High Availability Observability Stack.

MinIO object storage is excluded from the current phase and may be considered in a future phase.

## 8. Node Exporter Setup

### 8.1 Overview

Node Exporter is a Prometheus exporter that collects hardware and operating system metrics from Kubernetes nodes.

It collects metrics such as:

- CPU utilization
- Memory usage
- Disk usage
- Disk I/O
- Network traffic

These metrics are used to monitor the health and performance of Kubernetes nodes.

### 8.2 Deployment Details

| Configuration | Value |
|---|---|
| Component | Node Exporter |
| Namespace | metrics |
| Deployment Method | Kubernetes Manifest |
| Resource Type | DaemonSet |
| Image | quay.io/prometheus/node-exporter:v1.9.1 |
| Port | 9100 |
| Manifest Path | node-exporter/node-exporter.yaml |
| Storage | HostPath (Read-Only) |
| Deployment Status | Completed |

### 8.3 Directory Structure

```text
ha-observability-poc/
├── kind/
│   └── kind-config.yaml
├── node-exporter/
│   └── node-exporter.yaml
└── README.md
```

### 8.4 Node Exporter Manifest

File Path: `node-exporter/node-exporter.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: metrics
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
        - operator: Exists
      containers:
        - name: node-exporter
          image: quay.io/prometheus/node-exporter:v1.9.1
          args:
            - --path.rootfs=/host
          ports:
            - containerPort: 9100
              hostPort: 9100
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
          volumeMounts:
            - name: root
              mountPath: /host
              readOnly: true
      volumes:
        - name: root
          hostPath:
            path: /
            type: Directory
```

### 8.5 Configuration Explanation

| Configuration | Explanation |
|---|---|
| `kind: DaemonSet` | Runs one Node Exporter Pod on each eligible node. |
| `namespace: metrics` | Deploys Node Exporter in the metrics namespace. |
| `hostPID: true` | Allows the Pod to share the host PID namespace. |
| `hostNetwork: true` | Uses the host network namespace. |
| `tolerations` | Allows scheduling on nodes with taints. |
| `image` | Specifies the Node Exporter container image. |
| `containerPort: 9100` | Node Exporter metrics port. |
| `hostPort: 9100` | Exposes port 9100 on the node. |
| `resources.requests` | Defines the minimum requested CPU and memory. |
| `resources.limits` | Defines the maximum CPU and memory allowed. |
| `hostPath: /` | Mounts the node's root filesystem. |
| `mountPath: /host` | Mounts the host filesystem inside the container. |
| `readOnly: true` | Prevents the container from modifying the mounted filesystem. |

### 8.6 Deployment Commands

Validate the manifest:

```bash
kubectl apply --dry-run=client -f node-exporter/node-exporter.yaml
```

Deploy Node Exporter:

```bash
kubectl apply -f node-exporter/node-exporter.yaml
```

### 8.7 Verification Commands

Check the DaemonSet:

```bash
kubectl get daemonset -n metrics
```
<img width="896" height="82" alt="image" src="https://github.com/user-attachments/assets/8f3d36f7-191a-44bf-93b7-bd6775962954" />

Check the Pods:

```bash
kubectl get pods -n metrics -o wide
```
<img width="1233" height="119" alt="image" src="https://github.com/user-attachments/assets/fef80d89-1f4f-41b5-bd9e-6c6abc640392" />

Check the logs:

```bash
kubectl logs -n metrics -l app=node-exporter
```
<img width="1027" height="97" alt="image" src="https://github.com/user-attachments/assets/12d171e0-3159-4421-914c-52a178c02576" />

### 8.8 Deployment Result

Node Exporter was successfully deployed across all three Kind Kubernetes nodes.

| Resource | Expected Result | Actual Result |
|---|---|---|
| DaemonSet | 3/3 Ready | 3/3 Ready |
| Pods | 3 Running | 3 Running |
| Restarts | 0 | 0 |

### 8.9 Architecture

```text
Kind Kubernetes Cluster
         |
         +-- Control Plane Node
         |       |
         |   Node Exporter
         |
         +-- Worker Node 1
         |       |
         |   Node Exporter
         |
         +-- Worker Node 2
                 |
             Node Exporter

Node Exporter exposes metrics on port 9100.
vmagent will scrape these metrics in the next phase.
```
### 8.9 Metrics Endpoint Verification

Node Exporter exposes node-level metrics through an HTTP endpoint on port `9100`.

The metrics are available at the `/metrics` path.

#### Port Forwarding

```bash
kubectl port-forward -n metrics daemonset/node-exporter 9100:9100
```

#### Access Metrics

```bash
curl http://localhost:9100/metrics
```
<img width="1329" height="165" alt="image" src="https://github.com/user-attachments/assets/ac128bee-d9f0-4b7e-9ec9-e829e8ce0279" />

```

The `/metrics` endpoint returns metrics in Prometheus exposition format. These metrics will be scraped by vmagent and stored in VictoriaMetrics for visualization in Grafana.
