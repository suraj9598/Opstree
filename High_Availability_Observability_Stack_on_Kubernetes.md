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
```
## 9. Kube-State-Metrics Setup

### 9.1 Overview

Kube-state-metrics is a Kubernetes monitoring component that collects information about the state of Kubernetes objects and exposes it in Prometheus-compatible metrics format.

It provides metrics related to:
- Pods
- Nodes
- Deployments
- ReplicaSets
- StatefulSets
- PersistentVolumeClaims
- Jobs and CronJobs

Node Exporter collects operating-system-level metrics, whereas kube-state-metrics provides Kubernetes object-state metrics.

### 9.2 Installation Method

Kube-state-metrics is installed using Kubernetes YAML manifests.

**Manifest file location:**

`kube-state-metrics/kube-state-metrics.yaml`

**Namespace:** `metrics`

**Resources created:**
- ServiceAccount
- ClusterRole
- ClusterRoleBinding
- Deployment
- ClusterIP Service

### 9.3 Complete YAML Manifest

Create the file:

```bash
mkdir -p kube-state-metrics
nano kube-state-metrics/kube-state-metrics.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: metrics
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
      - services
      - persistentvolumeclaims
      - persistentvolumes
      - configmaps
      - replicationcontrollers
    verbs: ["list", "watch"]
  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
      - daemonsets
      - statefulsets
    verbs: ["list", "watch"]
  - apiGroups: ["batch"]
    resources:
      - jobs
      - cronjobs
    verbs: ["list", "watch"]
  - apiGroups: ["autoscaling"]
    resources:
      - horizontalpodautoscalers
    verbs: ["list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
      - networkpolicies
    verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: metrics
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
        - name: kube-state-metrics
          image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.20.0
          ports:
            - name: metrics
              containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /livez
              port: 8080
            initialDelaySeconds: 5
            timeoutSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: metrics
spec:
  type: ClusterIP
  selector:
    app: kube-state-metrics
  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
```

### 9.4 Installation Commands

Validate the manifest:

```bash
kubectl apply --dry-run=client \
  -f kube-state-metrics/kube-state-metrics.yaml
```

Deploy kube-state-metrics:

```bash
kubectl apply \
  -f kube-state-metrics/kube-state-metrics.yaml
```

### 9.5 Verification Commands

Check the Deployment:

```bash
kubectl get deployment -n metrics
```
<img width="947" height="57" alt="image" src="https://github.com/user-attachments/assets/3fa4ded9-5d45-4682-933b-0211324aaa5b" />

Check the Pod:

```bash
kubectl get pods -n metrics
```
<img width="943" height="112" alt="image" src="https://github.com/user-attachments/assets/da7430b5-602e-4dbb-a095-d613d0f87836" />

Check the Service:

```bash
kubectl get service -n metrics
```
<img width="1002" height="59" alt="image" src="https://github.com/user-attachments/assets/a31cc843-789c-4e09-9aed-83cee9c7b001" />

Check application logs:

```bash
kubectl logs -n metrics deployment/kube-state-metrics
```
<img width="1366" height="201" alt="image" src="https://github.com/user-attachments/assets/6ad46af5-7d30-464b-8775-d339d5cb54df" />

### 9.6 Metrics Endpoint Verification

Forward the Service port to the local machine:

```bash
kubectl port-forward \
  -n metrics service/kube-state-metrics 8080:8080
```

Verify Pod information metrics:

```bash
curl -s http://localhost:8080/metrics | grep kube_pod_info
```

Verify Node information metrics:

```bash
curl -s http://localhost:8080/metrics | grep kube_node_info
```

### 9.7 Verification Results

Kube-state-metrics was successfully deployed in the `metrics` namespace.

| Component | Status |
|---|---|
| Deployment | 1/1 READY |
| Pod | 1/1 Running |
| Restart Count | 0 |
| Service Type | ClusterIP |
| Metrics Port | 8080 |
| Kubernetes API Communication | Successful |
| Node Metrics | Successfully exposed |

The `kube_node_info` metric returned information for all three Kind cluster nodes:

- `ha-observability-control-plane`
- `ha-observability-worker`
- `ha-observability-worker2`

### 9.8 Role in the Observability Architecture

Kube-state-metrics exposes Kubernetes object-state metrics. vmagent will scrape these metrics and send them to VictoriaMetrics for storage and querying through Grafana.

## 10. VictoriaMetrics Setup

### 10.1 Overview

VictoriaMetrics is a time-series database used to store and query metrics collected from Kubernetes nodes and workloads.

In this POC, VictoriaMetrics Single is deployed with three replicas across the three nodes of the Kind Kubernetes cluster.

Each replica runs independently and has its own PersistentVolumeClaim (PVC). Pod Anti-Affinity is configured to distribute the replicas across different Kubernetes nodes.

The objective of this setup is to establish the metrics storage layer and prepare the infrastructure for evaluating High Availability (HA) scenarios.

**Current Status:**
- Three VictoriaMetrics replicas deployed successfully.
- All replicas are running on separate Kubernetes nodes.
- Three separate 5Gi PVCs are created and Bound.
- VictoriaMetrics Service discovers all three pod endpoints.
- VictoriaMetrics VMUI is accessible through the browser.
- vmagent scraping and remote-write configuration is pending.
- Metrics fan-out and failover testing are pending.

### 10.2 Why We Selected This Approach

#### 1. VictoriaMetrics Single

VictoriaMetrics Single was selected for this initial POC because it provides a lightweight and relatively simple deployment model.

It allows us to focus on Kubernetes scheduling, persistent storage, and independent replica behavior without introducing the additional complexity of a native VictoriaMetrics Cluster deployment.

#### 2. Three Replicas Across Three Nodes

We deployed three independent VictoriaMetrics Single instances across the control-plane node and two worker nodes.

This distribution allows us to evaluate pod and node failure scenarios and observe the availability of the remaining instances.

#### 3. Pod Anti-Affinity

Required Pod Anti-Affinity is configured to prevent matching VictoriaMetrics pods from running on the same Kubernetes node.

This ensures that each replica is distributed across a separate node.

#### 4. Separate Persistent Storage

Each VictoriaMetrics replica has its own 5Gi PersistentVolumeClaim.

This provides independent storage for each instance and avoids sharing the same volume between replicas.

#### 5. Helm-Based Deployment

Helm was selected to manage the deployment because it provides repeatable installation and centralized configuration through `values.yaml`.

Replica count, storage, resources, retention period, and scheduling rules can be maintained in a single configuration file.

#### 6. Kind for Local POC

Kind provides a local multi-node Kubernetes environment. It allows us to test the deployment and scheduling behavior without provisioning a separate production Kubernetes infrastructure.

**Important HA Limitation:** The three VictoriaMetrics Single instances do not automatically replicate data between themselves. The headless Service discovers the pod endpoints but does not distribute or replicate metrics data. vmagent remote-write fan-out and query failover must be configured and tested separately.

### 10.3 Deployment Details

| Configuration | Value |
|---|---|
| Kubernetes Cluster | `ha-observability` |
| Namespace | `metrics` |
| Helm Release | `vm-single` |
| Helm Chart | `vm/victoria-metrics-single` |
| Chart Version | `0.51.0` |
| VictoriaMetrics Version | `v1.156.0` |
| Deployment Type | StatefulSet |
| Replicas | 3 |
| Retention Period | 1 day |
| Storage per Replica | 5Gi |
| StorageClass | `standard` |
| HTTP Port | `8428` |

### 10.4 Helm Repository Setup

Add the VictoriaMetrics Helm repository:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
```

Update the Helm repository:

```bash
helm repo update
```

Check available chart versions:

```bash
helm search repo vm/victoria-metrics-single --versions
```

Create the metrics namespace:

```bash
kubectl create namespace metrics \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 10.5 VictoriaMetrics Configuration

**File Path:**

`~/ha-observability-poc/victoria-metrics/values.yaml`

```yaml
server:
  enabled: true
  replicaCount: 3

  retentionPeriod: "1"

  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: server
              app.kubernetes.io/instance: vm-single
              app.kubernetes.io/name: victoria-metrics-single
          topologyKey: kubernetes.io/hostname

  persistentVolume:
    enabled: true
    accessModes:
      - ReadWriteOnce
    storageClassName: standard
    size: 5Gi
    mountPath: /victoria-metrics-data

  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

### 10.6 Configuration Explanation

| Configuration | Explanation |
|---|---|
| `replicaCount: 3` | Creates three VictoriaMetrics Single replicas. |
| `retentionPeriod: "1"` | Configures one-day data retention. |
| `tolerations` | Allows a VictoriaMetrics pod to be scheduled on the control-plane node. |
| `podAntiAffinity` | Prevents matching pods from being scheduled on the same node. |
| `topologyKey: kubernetes.io/hostname` | Uses the node hostname as the scheduling separation boundary. |
| `persistentVolume.enabled: true` | Enables persistent storage for each replica. |
| `size: 5Gi` | Requests 5Gi storage for each replica. |
| `storageClassName: standard` | Uses the Kind `standard` StorageClass. |
| `resources.requests` | Defines the minimum CPU and memory requested by each pod. |
| `resources.limits` | Defines the maximum CPU and memory allowed for each pod. |

### 10.7 Render and Validate Helm Templates

Export the default Helm values:

```bash
helm show values vm/victoria-metrics-single \
  --version 0.51.0 > chart-default-values.yaml
```

Render the Helm templates:

```bash
helm template vm-single vm/victoria-metrics-single \
  --version 0.51.0 \
  --namespace metrics \
  --values values.yaml > vm-rendered.yaml
```

Verify the StatefulSet:

```bash
grep -n -A 45 'kind: StatefulSet' vm-rendered.yaml
```

Verify Anti-Affinity:

```bash
grep -n -A 35 'affinity:' vm-rendered.yaml
```

Verify PVC configuration:

```bash
grep -n -A 20 'volumeClaimTemplates:' vm-rendered.yaml
```

Verify tolerations:

```bash
grep -n -A 15 'tolerations:' vm-rendered.yaml
```

The rendered manifest was verified to contain three replicas, required Pod Anti-Affinity, control-plane toleration, and a 5Gi PVC template.

### 10.8 Install VictoriaMetrics

Install VictoriaMetrics using Helm:

```bash
helm upgrade --install vm-single vm/victoria-metrics-single \
  --version 0.51.0 \
  --namespace metrics \
  --values values.yaml
```

### 10.9 Verify VictoriaMetrics Pods

Check the pods and their node placement:

```bash
kubectl get pods -n metrics -o wide
```

**Observed Deployment Result:**

| Pod | Kubernetes Node | Status |
|---|---|---|
| `vm-single-victoria-metrics-single-server-0` | `ha-observability-worker2` | 1/1 Running |
| `vm-single-victoria-metrics-single-server-1` | `ha-observability-worker` | 1/1 Running |
| `vm-single-victoria-metrics-single-server-2` | `ha-observability-control-plane` | 1/1 Running |

All three VictoriaMetrics pods are running on separate Kubernetes nodes.

### 10.10 Verify PersistentVolumeClaims

Check PVCs:

```bash
kubectl get pvc -n metrics
```

**Observed Result:**

| PVC | Capacity | Status |
|---|---|---|
| `server-volume-vm-single-victoria-metrics-single-server-0` | 5Gi | Bound |
| `server-volume-vm-single-victoria-metrics-single-server-1` | 5Gi | Bound |
| `server-volume-vm-single-victoria-metrics-single-server-2` | 5Gi | Bound |

Each VictoriaMetrics replica has its own persistent storage.

### 10.11 Verify Service and Endpoints

Check the Service:

```bash
kubectl get svc -n metrics
```
<img width="1020" height="92" alt="image" src="https://github.com/user-attachments/assets/4e6fc637-e323-4b96-817a-35c2560f88d0" />

Check the Endpoints:

```bash
kubectl get endpoints -n metrics
```
<img width="1078" height="106" alt="image" src="https://github.com/user-attachments/assets/daf2eed2-450b-47f6-af14-4ad4e609bc64" />

**Observed Service Details:**

| Configuration | Value |
|---|---|
| Service Name | `vm-single-victoria-metrics-single-server` |
| Service Type | Headless ClusterIP |
| Cluster IP | `None` |
| Port | `8428` |
| Endpoints | 3 |

The headless Service discovers all three VictoriaMetrics pod endpoints on port `8428`.

This confirms endpoint discovery and does not indicate automatic data replication.

### 10.12 Access VictoriaMetrics UI in Browser

VictoriaMetrics VMUI can be accessed through Kubernetes port-forwarding.

Start port-forwarding:

```bash
kubectl port-forward -n metrics \
  svc/vm-single-victoria-metrics-single-server 8428:8428
```

Keep the terminal running.

Open the following URL in the browser:

```text
http://localhost:8428/vmui/
```
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/59a79357-43e8-44b1-9cbd-dfc463880d36" />

The VictoriaMetrics VMUI was successfully opened in the browser.

### 10.13 Query Verification

In the VMUI query section, execute the following query:

```promql
up
```

This query can be used to check the availability of scraped targets once metrics collection is configured.

At the current stage, the expected metrics may not be available because vmagent scraping and remote-write configuration are still pending.

### 10.14 Current Architecture

```text
                 Kind Kubernetes Cluster
                    ha-observability
                           |
          +----------------+----------------+
          |                |                |
    Control Plane      Worker Node 1    Worker Node 2
          |                |                |
  VictoriaMetrics    VictoriaMetrics   VictoriaMetrics
    Replica 2          Replica 1         Replica 0
          |                |                |
       PVC 5Gi           PVC 5Gi          PVC 5Gi
          |                |                |
          +----------------+----------------+
                           |
                Headless Service :8428
                           |
                         VMUI
```

