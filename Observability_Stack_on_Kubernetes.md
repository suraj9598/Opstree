# Observability Stack on Kubernetes

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



# 10. VictoriaMetrics Setup

## 10.1 Overview

VictoriaMetrics is a time-series database used to store and query metrics collected from Kubernetes nodes and workloads.

In this POC, VictoriaMetrics Single is deployed with three replicas across the three nodes of the Kind Kubernetes cluster.

Each replica runs independently and has its own PersistentVolumeClaim (PVC). Pod Anti-Affinity is configured to distribute the replicas across different Kubernetes nodes.

Initially, VictoriaMetrics was deployed as the metrics storage layer. Later, vmagent was configured to scrape metrics from Kubernetes exporters and forward metrics to all three VictoriaMetrics instances using remote-write.

The objective of this setup is to evaluate metrics collection, storage distribution, pod scheduling, and HA-related behavior.

### Current Status

- Three VictoriaMetrics Single replicas deployed successfully.
- All three replicas are running on separate Kubernetes nodes.
- Three separate 5Gi PVCs are created and Bound.
- VictoriaMetrics headless Service discovers all three pod endpoints.
- VictoriaMetrics VMUI is accessible through port-forwarding.
- vmagent is deployed with three replicas.
- vmagent scraping is configured for Node Exporter and kube-state-metrics.
- Remote-write is configured to send metrics to all three VictoriaMetrics endpoints.
- All three VictoriaMetrics endpoints successfully return metrics through query API.
- Remote-write errors and retries were observed as zero during the verification period.
- Failure testing and query failover testing are pending.

## 10.2 Why We Selected This Approach

### 1. VictoriaMetrics Single

VictoriaMetrics Single was selected for the initial POC because it provides a lightweight and relatively simple deployment model.

It allows us to focus on Kubernetes scheduling, persistent storage, and independent replica behavior without introducing the additional complexity of a native VictoriaMetrics Cluster deployment.

### 2. Three Replicas Across Three Nodes

Three VictoriaMetrics Single instances are deployed across the control-plane node and two worker nodes.

This distribution allows us to evaluate pod and node failure scenarios and observe the availability of the remaining instances.

### 3. Pod Anti-Affinity

Required Pod Anti-Affinity is configured to prevent matching VictoriaMetrics pods from running on the same Kubernetes node.

This ensures that each replica is distributed across a separate node.

If Anti-Affinity were not configured, Kubernetes could schedule multiple VictoriaMetrics replicas on the same node, reducing the benefit of node-level separation.

### 4. Separate Persistent Storage

Each VictoriaMetrics replica has its own 5Gi PersistentVolumeClaim.

This provides independent storage for each instance and avoids sharing the same volume between replicas.

Separate PVCs also allow each instance to maintain its own data independently.

### 5. Control-Plane Toleration

The Kind control-plane node has a `NoSchedule` taint.

A toleration is configured to allow VictoriaMetrics pods to run on the control-plane node.

Without this toleration, Kubernetes would not schedule the VictoriaMetrics pod on the control-plane node.

### 6. Helm-Based Deployment

Helm was selected to manage the deployment because it provides repeatable installation and centralized configuration through `values.yaml`.

Replica count, storage, resources, retention period, and scheduling rules can be maintained in a single configuration file.

### 7. Kind for Local POC

Kind provides a local multi-node Kubernetes environment.

It allows us to test deployment and scheduling behavior without provisioning separate production Kubernetes infrastructure.

### 8. vmagent Remote-Write Integration

Initially, the VictoriaMetrics instances were running independently and were not receiving metrics from vmagent.

To establish the metrics ingestion flow, vmagent was configured to scrape Kubernetes metrics and send them to the VictoriaMetrics endpoints.

This is necessary to evaluate metrics ingestion and data availability at each backend.

**Important HA Limitation:** The three VictoriaMetrics Single instances do not automatically replicate data between themselves. Remote-write fan-out sends metrics to the configured endpoints, but this is not native data replication between VictoriaMetrics servers. Query failover and failure scenarios must be tested separately.

## 10.3 Deployment Details

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

## 10.4 Helm Repository Setup

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

## 10.5 VictoriaMetrics Configuration

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

## 10.6 Configuration Explanation

| Configuration | Explanation | Why It Is Required |
|---|---|---|
| `replicaCount: 3` | Creates three VictoriaMetrics Single replicas. | To evaluate availability across three Kubernetes nodes. |
| `retentionPeriod: "1"` | Configures one-day data retention. | To control the duration of stored metrics during the POC. |
| `tolerations` | Allows a pod to run on the control-plane node. | The control-plane node has a `NoSchedule` taint. |
| `podAntiAffinity` | Prevents matching pods from running on the same node. | To distribute replicas across different nodes. |
| `topologyKey: kubernetes.io/hostname` | Uses node hostname as the separation boundary. | To ensure node-level distribution. |
| `persistentVolume.enabled: true` | Enables persistent storage. | To retain metrics data beyond the container lifecycle. |
| `size: 5Gi` | Requests 5Gi storage per replica. | To provide independent storage capacity. |
| `storageClassName: standard` | Uses the Kind `standard` StorageClass. | To provision local persistent volumes in Kind. |
| `resources.requests` | Defines minimum requested CPU and memory. | To help Kubernetes schedule the pods based on resource requirements. |
| `resources.limits` | Defines maximum CPU and memory allowed. | To prevent uncontrolled resource consumption. |

## 10.7 Render and Validate Helm Templates

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

## 10.8 Install VictoriaMetrics

Install VictoriaMetrics using Helm:

```bash
helm upgrade --install vm-single vm/victoria-metrics-single \
  --version 0.51.0 \
  --namespace metrics \
  --values values.yaml
```

## 10.9 Verify VictoriaMetrics Pods

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

## 10.10 Verify PersistentVolumeClaims

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

## 10.11 Verify Service and Endpoints

Check the Service:

```bash
kubectl get svc -n metrics
```

Check the Endpoints:

```bash
kubectl get endpoints -n metrics
```

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

## 10.12 Access VictoriaMetrics UI in Browser

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
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/ee7f53c3-9faf-4120-ab34-856794ad74d1" />


The VictoriaMetrics VMUI was successfully opened in the browser.

## 10.13 Query Verification

In the VMUI query section, execute:

```promql
up
```

This query checks the availability of scraped targets once metrics collection is configured.

After vmagent configuration, the query was also verified against the individual VictoriaMetrics instances.

## 10.14 Updated VictoriaMetrics Architecture

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
             Individual VMUI / Query API
```

The VictoriaMetrics instances run independently. vmagent sends metrics to the configured endpoints, but backend data replication is not automatic.

---

# 11. vmagent Setup

## 11.1 Overview

vmagent is a metrics collection and forwarding component developed by VictoriaMetrics.

It is responsible for scraping metrics from Prometheus-compatible endpoints and forwarding the collected metrics to VictoriaMetrics using the remote-write protocol.

In this POC, vmagent is deployed with three replicas in the `metrics` namespace.

The vmagent configuration includes:

- Three vmagent replicas.
- StatefulSet deployment mode.
- Cluster mode enabled.
- Replication factor set to 2.
- Global scrape interval of 10 seconds.
- Node Exporter scraping.
- kube-state-metrics scraping.
- Remote-write to all three VictoriaMetrics Single endpoints.
- Pod distribution across the three Kubernetes nodes.
- Control-plane toleration.

The objective is to create a distributed metrics collection and forwarding layer.

## 11.2 Why vmagent Was Required

Before vmagent was configured, VictoriaMetrics instances were deployed and their services were available, but the metrics collection and forwarding flow was not established.

vmagent was introduced to:

1. Collect metrics from Node Exporter.
2. Collect Kubernetes object-state metrics from kube-state-metrics.
3. Forward metrics to VictoriaMetrics.
4. Configure remote-write fan-out to the three VictoriaMetrics endpoints.
5. Evaluate distributed metrics collection.
6. Prepare the environment for future failure and HA testing.

## 11.3 Deployment Details

| Configuration | Value |
|---|---|
| Component | vmagent |
| Namespace | `metrics` |
| Helm Release | `vmagent` |
| Helm Chart | `vm/victoria-metrics-agent` |
| Chart Version | `0.52.0` |
| VictoriaMetrics App Version | `v1.156.0` |
| Replica Count | 3 |
| Deployment Mode | StatefulSet |
| Cluster Mode | Enabled |
| Replication Factor | 2 |
| Scrape Interval | 10 seconds |
| Remote-Write Backend | VictoriaMetrics Single |
| Manifest / Values File | `vmagent/vmagent-default-values.yaml` |

## 11.4 Helm Repository Setup

The VictoriaMetrics Helm repository was already configured during the VictoriaMetrics installation.

Verify the repository:

```bash
helm repo list
```

Update the repository:

```bash
helm repo update
```

Check the vmagent chart version:

```bash
helm search repo vm/victoria-metrics-agent --versions
```

## 11.5 vmagent Configuration File

**File Path:**

`~/ha-observability-poc/vmagent/vmagent-default-values.yaml`

The vmagent values file was customized to configure the replicas, cluster mode, remote-write destinations, scrape configuration, and scheduling rules.

### Core vmagent Configuration

The following represents the important configuration blocks used in the POC:

```yaml
replicaCount: 3

mode: statefulSet

statefulSet:
  clusterMode: true
  replicationFactor: 2

config:
  global:
    scrape_interval: 10s

  remoteWrite:
    - url: "http://vm-single-victoria-metrics-single-server-0.vm-single-victoria-metrics-single-server.metrics.svc.cluster.local:8428/api/v1/write"

    - url: "http://vm-single-victoria-metrics-single-server-1.vm-single-victoria-metrics-single-server.metrics.svc.cluster.local:8428/api/v1/write"

    - url: "http://vm-single-victoria-metrics-single-server-2.vm-single-victoria-metrics-single-server.metrics.svc.cluster.local:8428/api/v1/write"
```

The scrape configuration is also customized to discover Node Exporter and kube-state-metrics endpoints through Kubernetes service discovery.

> Note: The block above documents the key configuration values. The complete chart values file may contain additional chart-default fields and other settings.

## 11.6 Remote-Write Configuration Explanation

### 1. Three vmagent Replicas

```yaml
replicaCount: 3
```

This creates three vmagent pods.

**Why?**

To distribute metrics collection across the Kubernetes nodes and evaluate the behavior of multiple vmagent instances.

### 2. StatefulSet Mode

```yaml
mode: statefulSet
```

**Why?**

StatefulSet provides stable pod identities, which are useful when configuring distributed vmagent instances.

### 3. Cluster Mode

```yaml
statefulSet:
  clusterMode: true
```

**Why?**

Cluster mode enables vmagent's distributed scraping behavior so that scrape targets can be distributed across vmagent replicas rather than every replica independently scraping the complete target set.

### 4. Replication Factor

```yaml
statefulSet:
  replicationFactor: 2
```

**Why?**

The replication factor is configured to 2 to allow target scraping data to be replicated between vmagent instances according to its cluster-mode behavior.

This is a vmagent-side replication setting. It does not configure native replication between VictoriaMetrics Single servers.

### 5. Scrape Interval

```yaml
config:
  global:
    scrape_interval: 10s
```

**Why?**

Metrics are collected every 10 seconds.

A shorter scrape interval provides more frequent monitoring data but increases the number of scrape requests and the volume of metrics.

### 6. Remote-Write Endpoints

Remote-write is configured to send data to all three VictoriaMetrics Single endpoints.

**Why?**

Each VictoriaMetrics instance has its own storage. Sending metrics to the configured endpoints ensures that metrics are ingested by each configured destination.

This approach is used to evaluate metrics fan-out and backend availability.

## 11.7 Remote-Write Architecture

```text
                  Kubernetes Cluster
                 ha-observability
                         |
          +--------------+--------------+
          |              |              |
       Worker 1       Worker 2     Control Plane
          |              |              |
       vmagent-0      vmagent-1      vmagent-2
          |              |              |
          +--------------+--------------+
                         |
                  Remote Write
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
 VictoriaMetrics   VictoriaMetrics   VictoriaMetrics
   Replica 0         Replica 1         Replica 2
        |                |                |
      PVC 5Gi          PVC 5Gi          PVC 5Gi
```

This architecture shows the metrics collection and remote-write flow.

The VictoriaMetrics instances are independent storage backends. Remote-write fan-out is configured at vmagent.

## 11.8 Configure Scrape Targets

vmagent is configured to scrape metrics from the following components:

| Target | Namespace | Metrics Port | Purpose |
|---|---|---|---|
| Node Exporter | `metrics` | 9100 | Node CPU, memory, disk, and network metrics |
| kube-state-metrics | `metrics` | 8080 | Kubernetes Pod, Node, Deployment, and object-state metrics |

### Node Exporter

Node Exporter is deployed as a DaemonSet.

Each eligible Kubernetes node runs one Node Exporter pod, exposing metrics on port `9100`.

The vmagent Kubernetes service discovery configuration selects the Node Exporter service endpoints.

### kube-state-metrics

kube-state-metrics is deployed as a Deployment with one replica.

It exposes Kubernetes object-state metrics through its Service on port `8080`.

vmagent discovers the kube-state-metrics endpoint and scrapes the `/metrics` path.

## 11.9 Kubernetes Service Discovery

The vmagent scrape configuration uses Kubernetes endpoint service discovery.

The discovery configuration is restricted to the `metrics` namespace.

The service discovery configuration filters the relevant services and endpoint port name `metrics`.

**Why Kubernetes service discovery was used:**

- It automatically discovers endpoints when pods are recreated.
- It avoids manually maintaining static pod IP addresses.
- It supports dynamic Kubernetes workloads.
- It makes the scrape configuration easier to maintain.

## 11.10 Scheduling Configuration

vmagent pods are configured to distribute across the three Kind nodes.

### Control-Plane Toleration

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

**Why?**

The control-plane node has a `NoSchedule` taint. This toleration allows vmagent to run on the control-plane node.

Without the toleration, vmagent pods would be restricted from running on the control-plane node.

### Topology Spread Constraints

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/instance: vmagent
        app.kubernetes.io/name: victoria-metrics-agent
```

**Why?**

Topology spread constraints help distribute vmagent pods evenly across Kubernetes nodes.

`maxSkew: 1` allows a maximum difference of one matching pod between eligible nodes.

`topologyKey: kubernetes.io/hostname` uses the node hostname as the distribution boundary.

`whenUnsatisfiable: DoNotSchedule` prevents Kubernetes from scheduling a pod if the required distribution cannot be satisfied.

## 11.11 Install vmagent

Install vmagent using Helm:

```bash
helm upgrade --install vmagent vm/victoria-metrics-agent \
  --version 0.52.0 \
  --namespace metrics \
  --values vmagent/vmagent-default-values.yaml
```

## 11.12 Verify vmagent Pods

Check the pods:

```bash
kubectl get pods -n metrics -o wide
```

**Observed Deployment Result:**

| Pod | Kubernetes Node | Status |
|---|---|---|
| `vmagent-0` | `ha-observability-worker2` | Running |
| `vmagent-1` | `ha-observability-worker` | Running |
| `vmagent-2` | `ha-observability-control-plane` | Running |

All three vmagent replicas were observed running across the three Kubernetes nodes.

## 11.13 Verify vmagent Targets

vmagent exposes an HTTP endpoint for checking scrape target status.

Port-forward vmagent:

```bash
kubectl port-forward -n metrics pod/vmagent-0 8429:8429
```

Check target status:

```bash
curl -s http://localhost:8429/api/v1/targets
```

The target API provides information about discovered scrape targets and their health.

The configured targets were observed as healthy (`up`).

## 11.14 Verify Remote-Write Metrics

Check vmagent remote-write metrics:

```bash
curl -s http://localhost:8429/metrics | \
  grep -E 'vmagent_remotewrite_(errors|retries|requests)'
```

### Observed Verification

The following was observed during the verification period:

- Remote-write error counters were `0`.
- Remote-write retry counters were `0`.
- Successful HTTP 2XX remote-write requests were observed for all three configured destinations.
- The request counters were approximately 1471, 1466, and 1468 respectively at the time of observation.

These results indicate that vmagent successfully sent remote-write requests to the configured VictoriaMetrics endpoints during the observed period.

The counters are time-dependent and may change during subsequent runs.

## 11.15 Verify Metrics in All VictoriaMetrics Instances

Port-forward each VictoriaMetrics instance individually.

### VictoriaMetrics Replica 0

```bash
kubectl port-forward -n metrics \
  pod/vm-single-victoria-metrics-single-server-0 8428:8428
```

Query:

```bash
curl -s "http://localhost:8428/api/v1/query?query=up"
```

### VictoriaMetrics Replica 1

```bash
kubectl port-forward -n metrics \
  pod/vm-single-victoria-metrics-single-server-1 18429:8428
```

Query:

```bash
curl -s "http://localhost:18429/api/v1/query?query=up"
```

### VictoriaMetrics Replica 2

```bash
kubectl port-forward -n metrics \
  pod/vm-single-victoria-metrics-single-server-2 18430:8428
```

Query:

```bash
curl -s "http://localhost:18430/api/v1/query?query=up"
```

### Observed Result

All three VictoriaMetrics instances returned a successful query API response for `up`.

The returned metrics included:

- Node Exporter targets from the three Kind nodes.
- kube-state-metrics.
- Kubernetes API server.
- kubelet and cAdvisor-related targets.
- CoreDNS.
- vmagent.

This confirms that the metrics were available for querying in all three configured VictoriaMetrics endpoints during the verification period.

**Important:** Query results from the three instances do not prove native backend replication. Each backend remains an independent VictoriaMetrics Single instance.

## 11.16 Port-Forwarding Reference

The following local ports were used during verification:

| Component | Local Port | Remote Port |
|---|---:|---:|
| vmagent-0 | 8429 | 8429 |
| vmagent-1 | 8430 | 8429 |
| vmagent-2 | 8431 | 8429 |
| VictoriaMetrics Replica 0 | 8428 | 8428 |
| VictoriaMetrics Replica 1 | 18429 | 8428 |
| VictoriaMetrics Replica 2 | 18430 | 8428 |

Port-forwarding is used only for local verification and browser/API access.

## 11.17 Current Metrics Ingestion Architecture

```text
                     Kind Kubernetes Cluster
                       ha-observability
                               |
               +---------------+---------------+
               |               |               |
         Control Plane      Worker Node 1    Worker Node 2
               |               |               |
         Node Exporter    Node Exporter    Node Exporter
               |               |               |
               +---------------+---------------+
                               |
                       Kubernetes Discovery
                               |
                          vmagent
                    (3 StatefulSet Replicas)
                               |
                         Remote-Write
                               |
          +--------------------+--------------------+
          |                    |                    |
          v                    v                    v
 VictoriaMetrics         VictoriaMetrics       VictoriaMetrics
   Replica 0               Replica 1             Replica 2
     PVC 5Gi                 PVC 5Gi               PVC 5Gi
          |                    |                    |
          +--------------------+--------------------+
                               |
                      Query API / VMUI
                               |
                            Grafana
```

## 11.18 HA Considerations and Limitations

The current POC provides distributed pod placement and metrics ingestion to multiple independent VictoriaMetrics Single instances.

However, the following limitations must be considered:

1. **No native VictoriaMetrics Single data replication:** Each instance maintains its own storage.
2. **Remote-write fan-out is not backend replication:** Metrics are sent to configured endpoints independently.
3. **Query failover is not configured:** A query client must be configured to handle backend availability.
4. **Failure testing is pending:** Backend and vmagent failure scenarios have not yet been tested.
5. **Kind is a local POC environment:** The setup does not represent production infrastructure availability.
6. **Storage is independent:** A PVC failure affects the data stored on that particular VictoriaMetrics instance.
7. **Data consistency and duplicate handling:** These behaviors must be evaluated during further testing.

## 11.19 Access vmagent Through Browser

vmagent provides a web interface that can be accessed through a browser to check scrape targets and internal metrics.

Since vmagent is deployed inside the Kubernetes cluster in the `metrics` namespace, we use Kubernetes port-forwarding to access it locally.

### 11.19.1 Port-Forward vmagent

First, check the vmagent pods:

```bash
kubectl get pods -n metrics -o wide
```

Forward the vmagent port to the local machine:

```bash
kubectl port-forward -n metrics pod/vmagent-0 8429:8429
```

Keep this terminal running while accessing vmagent through the browser.

### 11.19.2 Open vmagent in Browser

Open the following URLs in your browser:

| URL | Description |
|---|---|
| http://localhost:8429 | vmagent web interface |
| http://localhost:8429/targets | View scrape targets and their status |
| http://localhost:8429/metrics | View internal vmagent metrics |

### 11.19.3 Access Other vmagent Replicas

vmagent is deployed with 3 replicas. Each replica can be accessed using a separate local port.

| vmagent Pod | Port-Forward Command | Browser URL |
|---|---|---|
| vmagent-0 | `kubectl port-forward -n metrics pod/vmagent-0 8429:8429` | http://localhost:8429 |
| vmagent-1 | `kubectl port-forward -n metrics pod/vmagent-1 8430:8429` | http://localhost:8430 |
| vmagent-2 | `kubectl port-forward -n metrics pod/vmagent-2 8431:8429` | http://localhost:8431 |

Run each port-forward command in a separate terminal.

### 11.19.4 Verify Scrape Targets

Open the following URL:

```text
http://localhost:8429/targets
```

This page displays the scrape targets discovered by vmagent along with their health status.

Targets showing `UP` indicate that vmagent is successfully scraping metrics from the respective endpoints.

## 12. Loki Setup

### 12.1 Overview

Grafana Loki is a log aggregation system used to collect, store, and query logs from Kubernetes workloads.

In this POC, Loki is deployed in the `logging` namespace using the Grafana Helm chart.

Loki is configured in SingleBinary mode with filesystem-based storage backed by a Kubernetes PersistentVolumeClaim (PVC).

The objective of this setup is to establish the log storage layer and prepare the infrastructure for log collection through Fluent Bit and visualization through Grafana.

**Current Status:**
- Loki deployed successfully using Helm.
- SingleBinary mode configured with one replica.
- Filesystem storage configured.
- A 5Gi PersistentVolumeClaim is created and Bound.
- Loki readiness endpoint returns `ready`.
- Loki build information API is accessible.
- Loki Kubernetes ClusterIP Service is created.
- MinIO is disabled in the current configuration.
- Fluent Bit log forwarding and Grafana datasource integration are pending.

### 12.2 Why We Selected This Approach

#### 1. SingleBinary Mode

SingleBinary mode runs Loki's required components together in a single instance.

This approach was selected because the current POC runs on a local Kind cluster with limited system resources.

It provides a simple deployment model and reduces the number of pods required for Loki.

#### 2. One Replica

Loki is configured with one replica:

```yaml
deploymentMode: SingleBinary

singleBinary:
  replicas: 1
```

One replica is sufficient for validating Loki installation, log ingestion, storage, and querying.

**Important HA Limitation:** SingleBinary with one replica does not provide high availability. If the Loki pod becomes unavailable, log ingestion and querying may be interrupted until the pod recovers.

#### 3. Filesystem Storage

Filesystem storage was selected for this POC.

Loki stores its log data on the filesystem backed by a Kubernetes PVC.

This avoids the need for an external object storage system during the initial setup.

#### 4. MinIO Disabled

MinIO is an S3-compatible object storage system that can be used as a storage backend for Loki.

During the initial setup, MinIO image pulls failed with a `401 UNAUTHORIZED` error.

Since the POC uses filesystem storage, MinIO is not required.

MinIO was therefore disabled to:
- Reduce resource consumption.
- Avoid additional pods and services.
- Simplify storage configuration.
- Avoid the encountered image pull issue.

MinIO is not mandatory for every Loki deployment. It is one possible object storage backend.

#### 5. Persistent Storage

A 5Gi PVC is configured for Loki.

This provides persistent storage for Loki data across pod restarts, provided the underlying volume is retained.

#### 6. Helm-Based Deployment

Helm was selected to manage Loki because it provides repeatable installation and configuration through a centralized `values.yaml` file.

Deployment mode, replicas, storage, resources, and optional components can be maintained in a single configuration file.

### 12.3 Deployment Details

| Configuration | Value |
|---|---|
| Kubernetes Cluster | `ha-observability` |
| Namespace | `logging` |
| Helm Release | `loki` |
| Helm Repository | `grafana` |
| Helm Chart | `grafana/loki` |
| Chart Version | `7.6.0` |
| Loki Image Version | `3.6.11` |
| Deployment Mode | SingleBinary |
| Resource Type | StatefulSet |
| Replicas | 1 |
| Storage Backend | Filesystem |
| Persistent Storage | 5Gi |
| StorageClass | `standard` |
| Loki HTTP Port | `3100` |
| Loki gRPC Port | `9095` |
| MinIO | Disabled |

### 12.4 Helm Repository Setup

Add the Grafana Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update the repository:

```bash
helm repo update grafana
```

Check available Loki chart versions:

```bash
helm search repo grafana/loki --versions
```

For this POC, chart version `7.6.0` was selected.

### 12.5 Loki Configuration

**Directory Path:**

`~/ha-observability-poc/loki`

Create the directory:

```bash
mkdir -p ~/ha-observability-poc/loki
cd ~/ha-observability-poc/loki
```

Create the configuration file:

```bash
nano loki-values.yaml
```

**File Path:**

`loki/loki-values.yaml`

#### Complete Loki Values YAML

```yaml
deploymentMode: SingleBinary

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  storage:
    type: filesystem

singleBinary:
  replicas: 1

  persistence:
    enabled: true
    storageClass: standard
    size: 5Gi

  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

minio:
  enabled: false

gateway:
  enabled: false

read:
  replicas: 0

write:
  replicas: 0

backend:
  replicas: 0

chunksCache:
  enabled: false

resultsCache:
  enabled: false
```

### 12.6 Configuration Explanation

| Configuration | Explanation |
|---|---|
| `deploymentMode: SingleBinary` | Runs Loki components together in one instance. |
| `auth_enabled: false` | Disables multi-tenant authentication for this POC. |
| `replication_factor: 1` | Configures one copy of data for the single-instance setup. |
| `store: tsdb` | Uses TSDB for Loki's index storage. |
| `object_store: filesystem` | Stores Loki data on the filesystem. |
| `schema: v13` | Defines the Loki storage schema version. |
| `singleBinary.replicas: 1` | Creates one Loki replica. |
| `persistence.enabled: true` | Enables persistent storage. |
| `storageClass: standard` | Uses the Kind `standard` StorageClass. |
| `size: 5Gi` | Requests 5Gi storage for Loki. |
| `resources.requests` | Defines minimum requested CPU and memory. |
| `resources.limits` | Defines maximum CPU and memory allowed. |
| `minio.enabled: false` | Disables MinIO object storage. |
| `gateway.enabled: false` | Disables the gateway component for this setup. |
| `read.replicas: 0` | Disables separate read components in SingleBinary mode. |
| `write.replicas: 0` | Disables separate write components in SingleBinary mode. |
| `backend.replicas: 0` | Disables separate backend components in SingleBinary mode. |
| `chunksCache.enabled: false` | Disables the chunks cache to reduce resource usage. |
| `resultsCache.enabled: false` | Disables the results cache to reduce resource usage. |

### 12.7 Render and Validate Helm Templates

Render the Helm templates before installation:

```bash
helm template loki grafana/loki \
  --namespace logging \
  --version 7.6.0 \
  -f loki-values.yaml \
  > loki-rendered.yaml
```

Verify the StatefulSet:

```bash
grep -n "kind: StatefulSet" loki-rendered.yaml
```

Verify the PersistentVolumeClaim:

```bash
grep -n "kind: PersistentVolumeClaim" loki-rendered.yaml
```

The rendered output was checked to confirm the SingleBinary StatefulSet and persistent storage configuration.

### 12.8 Install Loki

Install Loki using Helm:

```bash
helm install loki grafana/loki \
  --namespace logging \
  --create-namespace \
  --version 7.6.0 \
  -f loki-values.yaml
```

Check the Helm release:

```bash
helm list -n logging
```

Check the release status:

```bash
helm status loki -n logging
```

### 12.9 Verify Loki Pods

Check the pods:

```bash
kubectl get pods -n logging -o wide
```

Describe the Loki pod:

```bash
kubectl describe pod loki-0 -n logging
```

Check Loki application logs:

```bash
kubectl logs loki-0 -n logging -c loki
```

The Loki pod contains:
- Loki main container.
- Loki rules sidecar container.

The pod becomes Ready when both containers are ready.

### 12.10 Verify PersistentVolumeClaim

Check PVCs:

```bash
kubectl get pvc -n logging
```

**Observed Result:**

| PVC | Capacity | Status | StorageClass |
|---|---|---|---|
| `storage-loki-0` | 5Gi | Bound | `standard` |

The PVC is successfully Bound and attached to the Loki pod.

### 12.11 Verify Loki Services

Check the Services:

```bash
kubectl get svc -n logging
```

**Observed Services:**

| Service | Type | Port | Purpose |
|---|---|---|---|
| `loki` | ClusterIP | 3100, 9095 | Loki API and gRPC |
| `loki-canary` | ClusterIP | 3500 | Loki Canary |
| `loki-headless` | Headless | 3100 | Pod discovery |
| `loki-memberlist` | Headless | 7946 | Memberlist communication |

Loki internal service URL:

```text
http://loki.logging.svc.cluster.local:3100
```

This URL can be used by applications running inside the Kubernetes cluster.

### 12.12 Loki Port Forwarding

Forward Loki's port to the local machine:

```bash
kubectl port-forward -n logging pod/loki-0 3100:3100
```

Keep the terminal running.

Loki can now be accessed locally using:

```text
http://localhost:3100
```

### 12.13 Readiness Endpoint Verification

Execute:

```bash
curl http://localhost:3100/ready
```

**Observed Response:**

```text
ready
```

Verbose verification:

```bash
curl -v http://localhost:3100/ready
```

**Observed HTTP Response:**

```text
HTTP/1.1 200 OK
```

This confirms that Loki is ready and responding through its HTTP endpoint.

### 12.14 Loki Build Information Verification

Execute:

```bash
curl http://localhost:3100/loki/api/v1/status/buildinfo
```

**Observed Response:**

```json
{
  "version": "3.6.11",
  "revision": "f7a4aa99",
  "branch": "release-3.6.x",
  "buildUser": "root@buildkitsandbox",
  "buildDate": "2026-05-13T07:55:25Z",
  "goVersion": ""
}
```

This confirms that the Loki API is accessible and returns build information.

### 12.15 Troubleshooting and Changes Made

#### MinIO Image Pull Failure

During the initial installation attempt, MinIO and mc image pulls failed with:

```text
401 UNAUTHORIZED
```

**Resolution:**

MinIO was disabled in the Loki Helm values file:

```yaml
minio:
  enabled: false
```

Loki was configured to use filesystem storage backed by the PVC.

#### Initial Readiness Probe Failure

During Loki startup, the readiness probe temporarily returned HTTP 503.

The pod later became Ready.

**Verification Command:**

```bash
kubectl describe pod loki-0 -n logging
```

#### Optional Internal Connectivity Test

A temporary curl test pod was created to test internal service connectivity.

The pod terminated with an Error status. The exact cause was not established, and this optional test was skipped.

Loki readiness and build information were successfully verified using port-forwarding.

### 12.16 Final Deployment Result

| Component | Expected Result | Observed Result |
|---|---|---|
| Helm Release | Deployed | Deployed |
| Loki Pod | Running and Ready | Ready |
| Loki Containers | 2/2 Ready | 2/2 Ready |
| PVC | Bound | Bound |
| Storage Capacity | 5Gi | 5Gi |
| Readiness Endpoint | `ready` | `ready` |
| HTTP Readiness Status | 200 OK | 200 OK |
| Build Info API | Accessible | Accessible |
| MinIO | Disabled | Disabled |

### 12.17 Role in the Observability Architecture

Loki is the log storage component of the observability stack.

The intended log flow is:

```text
Kubernetes Workloads
        |
        |
Container Logs
        |
        |
    Fluent Bit
        |
        |
       Loki
        |
        |
      Grafana
  Log Visualization
```

Fluent Bit will collect logs from Kubernetes workloads and forward them to Loki.

Grafana will use Loki as a data source to query and visualize the collected logs.

# 13. Fluent Bit Setup

## 13.1 Overview

Fluent Bit is a lightweight log processor and forwarder used to collect logs from Kubernetes workloads and forward them to a centralized log storage system.

In this POC, Fluent Bit is deployed as a DaemonSet in the `logging` namespace of the `ha-observability` Kind Kubernetes cluster.

Fluent Bit collects container logs from Kubernetes nodes, enriches them with Kubernetes metadata, and forwards them to Loki using the Loki output plugin.

The objective of this setup is to establish the complete Kubernetes log collection and forwarding pipeline.

**Log Flow:**

`Kubernetes Container Logs → Fluent Bit → Loki → LogQL Queries`

### Current Status

- Fluent Bit deployed successfully using Helm.
- Fluent Bit deployed as a DaemonSet.
- Three Fluent Bit pods running across all three Kubernetes nodes.
- Control-plane toleration configured.
- Container logs collected from `/var/log/containers/`.
- Kubernetes metadata attached to log records.
- Loki output configured using the Loki service DNS name.
- Initial DNS resolution issue investigated and addressed through a hostname configuration change.
- Fluent Bit logs successfully forwarded to Loki.
- Kubernetes container logs verified through the Loki Query API.
- Fluent Bit's own container logs also verified in Loki.
- LogQL-compatible query execution through Loki API verified.

## 13.2 Why Fluent Bit Was Required

Loki is responsible for storing and querying logs, but it does not independently collect container log files from every Kubernetes node.

A log collection agent is required to read logs from the nodes and forward them to Loki.

Fluent Bit was selected for the following reasons:

### 1. Lightweight Log Collector

Fluent Bit is designed to collect and forward logs with relatively low resource consumption.

It is suitable for Kubernetes environments where a log collector needs to run on every node.

### 2. DaemonSet Deployment

Fluent Bit is deployed as a DaemonSet.

A DaemonSet ensures that one Fluent Bit pod runs on every eligible Kubernetes node.

In this POC, the cluster contains:

- One control-plane node.
- Two worker nodes.

Therefore, three Fluent Bit pods are deployed.

### 3. Kubernetes Metadata Enrichment

Fluent Bit uses the Kubernetes filter to enrich collected logs with metadata.

Examples include:

- Namespace name.
- Pod name.
- Container name.

This metadata helps identify which Kubernetes workload generated a particular log.

### 4. Loki Integration

Fluent Bit supports a Loki output plugin.

This plugin allows Fluent Bit to forward collected log records directly to Loki over HTTP.

### 5. Helm-Based Deployment

Helm is used to install and manage Fluent Bit.

This makes the deployment repeatable and allows configuration changes through a centralized values file.

## 13.3 Deployment Details

| Configuration | Value |
|---|---|
| Kubernetes Cluster | `ha-observability` |
| Namespace | `logging` |
| Component | Fluent Bit |
| Helm Release | `fluent-bit` |
| Helm Repository | `fluent` |
| Helm Chart | `fluent/fluent-bit` |
| Chart Version | `0.58.3` |
| Fluent Bit Application Version | `5.1.3` |
| Deployment Type | DaemonSet |
| Replicas / Pods | 3 |
| Input Plugin | Tail |
| Log Path | `/var/log/containers/*.log` |
| Filter | Kubernetes |
| Output Plugin | Loki |
| Loki Service | `loki.logging.svc.cluster.local.` |
| Loki Port | `3100` |
| Output Match | `kube.*` |

## 13.4 Fluent Bit Architecture

The following architecture represents the complete log collection and forwarding flow.

```text
                   Kind Kubernetes Cluster
                      ha-observability
                              |
              +---------------+---------------+
              |               |               |
              |               |               |
        Control Plane      Worker Node 1    Worker Node 2
              |               |               |
         Container Logs  Container Logs  Container Logs
              |               |               |
              v               v               v
         Fluent Bit      Fluent Bit      Fluent Bit
           Pod              Pod              Pod
              |               |               |
              +---------------+---------------+
                              |
                       Kubernetes Filter
                              |
                    Metadata Enrichment
                              |
                         Loki Output
                              |
                              v
                    Loki Service :3100
                              |
                              v
                       Loki Storage
                              |
                              v
                     Loki Query API
                              |
                              v
                         Grafana
```

### Architecture Flow Explanation

1. Kubernetes workloads generate container logs.
2. Container logs are written to the node's `/var/log/containers/` directory.
3. Fluent Bit runs as a DaemonSet on every eligible node.
4. The Tail input plugin reads container log files.
5. The Kubernetes filter extracts and attaches Kubernetes metadata.
6. The Loki output plugin forwards matching log records to Loki.
7. Loki receives and stores the log streams.
8. Users can query the collected logs using LogQL through Loki's API or Grafana.

## 13.5 Directory Structure

The Fluent Bit configuration is maintained separately inside the project directory.

```text
ha-observability-poc/
│
├── kind/
│   └── kind-config.yaml
│
├── node-exporter/
│   └── node-exporter.yaml
│
├── kube-state-metrics/
│   └── kube-state-metrics.yaml
│
├── victoria-metrics/
│   └── values.yaml
│
├── vmagent/
│   └── vmagent-default-values.yaml
│
├── loki/
│   └── loki-values.yaml
│
├── fluent-bit/
│   └── fluent-bit-values.yaml
│
└── README.md
```

## 13.6 Helm Repository Setup

Fluent Bit is installed using the official Fluent Bit Helm chart.

Add the Helm repository:

```bash
helm repo add fluent https://fluent.github.io/helm-charts
```

Update the Helm repository:

```bash
helm repo update
```

Verify the repository:

```bash
helm repo list
```

Check available Fluent Bit chart versions:

```bash
helm search repo fluent/fluent-bit --versions
```

For this POC, chart version `0.58.3` was selected.

## 13.7 Fluent Bit Configuration

### Configuration File

**File Path:**

`~/ha-observability-poc/fluent-bit/fluent-bit-values.yaml`

Create the directory:

```bash
mkdir -p ~/ha-observability-poc/fluent-bit
cd ~/ha-observability-poc/fluent-bit
```

Create the values file:

```bash
nano fluent-bit-values.yaml
```

### Configuration Used in the POC

```yaml
kind: DaemonSet

tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule

config:
  outputs: |
    [OUTPUT]
        Name                  loki
        Match                 kube.*
        Host                  loki.logging.svc.cluster.local.
        Port                  3100
        Labels                job=fluent-bit
        Label_Keys            $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name']
        Auto_Kubernetes_Labels Off
        Retry_Limit           False
```

**Important:** The above is the custom values configuration used for this POC. The Fluent Bit Helm chart also supplies its default input and filter configuration.

## 13.8 Configuration Explanation

### 1. DaemonSet

```yaml
kind: DaemonSet
```

Fluent Bit is deployed as a DaemonSet.

This ensures that one Fluent Bit pod runs on each eligible Kubernetes node.

Since the Kind cluster has three nodes, the expected Fluent Bit pod count is three.

### 2. Control-Plane Toleration

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

The control-plane node has a `NoSchedule` taint.

Without the required toleration, Fluent Bit would not be scheduled on that node.

The toleration allows Fluent Bit to run on the control-plane node as well as the worker nodes.

### 3. Loki Output Plugin

```ini
[OUTPUT]
    Name loki
```

The `loki` output plugin is used to forward collected log records to Loki.

### 4. Match

```ini
Match kube.*
```

The Match parameter determines which Fluent Bit records are sent to this output.

The configured output matches records with tags beginning with `kube.`.

The Tail input uses the Kubernetes log tag pattern, allowing matching Kubernetes container logs to be forwarded to Loki.

### 5. Loki Host

```ini
Host loki.logging.svc.cluster.local.
```

This is the internal Kubernetes DNS name of the Loki service.

Breaking down the hostname:

| Part | Meaning |
|---|---|
| `loki` | Loki Service name |
| `logging` | Kubernetes namespace |
| `svc` | Kubernetes Service DNS domain |
| `cluster.local` | Kubernetes cluster DNS domain |
| Final `.` | Marks the hostname as an absolute DNS name |

The trailing dot is intentional. It was added during DNS troubleshooting to avoid the observed DNS search-domain expansion behavior.

### 6. Loki Port

```ini
Port 3100
```

Loki exposes its HTTP API on port `3100`.

Fluent Bit sends logs to this port.

### 7. Labels

```ini
Labels job=fluent-bit
```

This adds a static Loki label:

`job="fluent-bit"`

It helps identify log streams forwarded by this Fluent Bit output.

### 8. Label_Keys

```ini
Label_Keys $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name']
```

This configuration extracts Kubernetes metadata fields and uses them as Loki labels.

The labels include:

- `kubernetes_namespace_name`
- `kubernetes_pod_name`
- `kubernetes_container_name`

These labels make it easier to filter logs by namespace, pod, and container.

### 9. Auto_Kubernetes_Labels

```ini
Auto_Kubernetes_Labels Off
```

Automatic conversion of all Kubernetes metadata labels into Loki labels is disabled.

Only the explicitly configured label keys are selected.

This keeps the label selection controlled instead of automatically adding every Kubernetes label.

### 10. Retry_Limit

```ini
Retry_Limit False
```

This configures unlimited retries for failed output delivery.

It allows Fluent Bit to continue retrying when Loki is temporarily unavailable, subject to Fluent Bit's buffering and retry behavior.

## 13.9 Input and Filter Configuration

The Fluent Bit chart provides the default input and filter configuration used in this POC.

### Tail Input

The Tail input plugin reads Kubernetes container log files.

The relevant configuration is:

```ini
[INPUT]
    Name tail
    Path /var/log/containers/*.log
    Tag kube.*
    Multiline.Parser docker, cri
    Mem_Buf_Limit 5MB
    Skip_Long_Lines On
```

**Explanation:**

| Parameter | Purpose |
|---|---|
| `Name tail` | Uses the Tail input plugin. |
| `Path` | Reads container log files from the node. |
| `Tag kube.*` | Assigns Kubernetes-related tags to collected records. |
| `Multiline.Parser` | Supports Docker and CRI multiline log parsing. |
| `Mem_Buf_Limit` | Sets the configured memory buffer limit for the input. |
| `Skip_Long_Lines` | Skips log lines exceeding the supported buffer conditions rather than stopping collection. |

### Kubernetes Filter

The Kubernetes filter enriches log records with Kubernetes metadata.

The chart's default configuration includes settings such as:

```ini
[FILTER]
    Name kubernetes
    Match kube.*
    Merge_Log On
    Keep_Log Off
```

**Explanation:**

| Parameter | Purpose |
|---|---|
| `Name kubernetes` | Enables Kubernetes metadata enrichment. |
| `Match kube.*` | Applies the filter to Kubernetes log records. |
| `Merge_Log On` | Attempts to merge structured JSON log content into the log record. |
| `Keep_Log Off` | Removes the original log field after successful merging according to the filter behavior. |

The Kubernetes filter helps attach useful metadata to log records before they are forwarded to Loki.

## 13.10 Deploy Fluent Bit

### Step 1: Validate Helm Configuration

Before installation, render the Helm templates:

```bash
helm template fluent-bit fluent/fluent-bit \
  --namespace logging \
  --version 0.58.3 \
  -f fluent-bit-values.yaml \
  > fluent-bit-rendered.yaml
```

This helps verify the generated Kubernetes resources before deployment.

### Step 2: Install Fluent Bit

```bash
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --version 0.58.3 \
  -f fluent-bit-values.yaml
```

If Fluent Bit is already installed and the configuration needs to be updated, use:

```bash
helm upgrade fluent-bit fluent/fluent-bit \
  --namespace logging \
  --version 0.58.3 \
  -f fluent-bit-values.yaml
```

### Step 3: Verify Helm Release

```bash
helm list -n logging
```

Check the release status:

```bash
helm status fluent-bit -n logging
```

## 13.11 Verify Fluent Bit Deployment

### Check DaemonSet

```bash
kubectl get daemonset -n logging
```

The DaemonSet should show three desired, current, and ready pods.

### Check Fluent Bit Pods

```bash
kubectl get pods -n logging \
  -l app.kubernetes.io/name=fluent-bit \
  -o wide
```

**Observed Deployment Result:**

| Node | Fluent Bit Pod Status |
|---|---|
| `ha-observability-control-plane` | Running |
| `ha-observability-worker` | Running |
| `ha-observability-worker2` | Running |

All three Fluent Bit pods were observed in `1/1 Running` state.

### Check Fluent Bit Logs

```bash
kubectl logs -n logging \
  -l app.kubernetes.io/name=fluent-bit \
  --prefix
```

This command displays the logs from all Fluent Bit pods.

It can be used to inspect startup messages, input activity, and output delivery errors.

### Check Helm Values

```bash
helm get values fluent-bit -n logging -o yaml
```

This command displays the custom values applied to the Helm release.

## 13.12 Troubleshooting Performed During Fluent Bit Setup

During the initial deployment, Fluent Bit pods were running, but logs were not being forwarded successfully to Loki.

The main issue observed during troubleshooting was DNS resolution failure from Fluent Bit.

### Issue 1: Loki DNS Resolution Failure

**Observed Error:**

```text
getaddrinfo(host='loki.logging.svc.cluster.local', err=11):
Could not contact DNS servers

no upstream connections available
```

This indicated that Fluent Bit was unable to resolve the Loki service hostname during the affected attempts.

As a result, Fluent Bit could not establish the required connection to Loki.

#### Step 1: Verify Loki Service

The Loki service was checked first:

```bash
kubectl get svc -n logging
```

The Loki service was available as a ClusterIP service on port `3100`.

The service name was:

```text
loki
```

The expected internal DNS name was:

```text
loki.logging.svc.cluster.local
```

#### Step 2: Check CoreDNS Pods

CoreDNS pods were checked to verify that the cluster DNS component was running:

```bash
kubectl get pods -n kube-system
```

The CoreDNS pods were in Running state.

#### Step 3: Check CoreDNS Service and Endpoints

The DNS service was inspected:

```bash
kubectl get svc -n kube-system kube-dns
```

The service was available with the ClusterIP:

```text
10.96.0.10
```

The DNS endpoints were also checked:

```bash
kubectl get endpoints -n kube-system kube-dns
```

This confirmed that CoreDNS endpoints were available.

#### Step 4: Inspect CoreDNS Logs

CoreDNS logs were checked:

```bash
kubectl logs -n kube-system \
  -l k8s-app=kube-dns \
  --prefix
```

During the investigation, upstream DNS timeout messages were observed.

An example of the upstream DNS server involved was:

```text
172.19.0.1:53
```

Queries were also observed with additional search-domain suffixes, such as:

```text
loki.logging.svc.cluster.local.netbird.selfhosted.
```

This indicated that DNS search-domain expansion was part of the observed DNS behavior.

#### Step 5: Inspect Pod DNS Configuration

A temporary DNS test pod was used to inspect the resolver configuration.

The resolver file contained:

```text
search logging.svc.cluster.local svc.cluster.local cluster.local netbird.selfhosted tail5caa65.ts.net

nameserver 10.96.0.10

options ndots:5
```

**Explanation:**

| Configuration | Purpose |
|---|---|
| `search` | Defines DNS suffixes used during hostname resolution. |
| `nameserver` | Specifies the DNS server used by the pod. |
| `ndots:5` | Controls when a name is treated as fully qualified versus being tried with search suffixes. |

The DNS configuration helped investigate why hostname resolution was behaving differently than expected.

### Issue 2: Verify DNS Resolution from Different Nodes

Temporary DNS test pods were created on different worker nodes.

The purpose was to determine whether Loki service DNS resolution worked from the Kubernetes node network.

DNS resolution was tested from:

- Worker node 1.
- Worker node 2.

The Loki service name resolved successfully from these test pods.

This helped establish that DNS resolution was possible from other pods in the cluster.

### Issue 3: Verify DNS from Fluent Bit Pod Network

Since DNS resolution worked from the temporary test pods, the next step was to test DNS from the Fluent Bit pod's network namespace.

An ephemeral debug container was added to a Fluent Bit pod using Kubernetes debug functionality.

The debug container was used to test DNS resolution and HTTP connectivity without modifying the Fluent Bit application configuration.

DNS resolution of the Loki service was successful from the debug container.

### Issue 4: Verify Loki HTTP Connectivity

After DNS resolution, Loki's HTTP readiness endpoint was tested from the Fluent Bit pod's network namespace.

The following command was executed inside the debug container:

```bash
wget -qO- http://loki.logging.svc.cluster.local:3100/ready
```

**Observed Response:**

```text
ready
```

This confirmed that the Loki service was reachable over HTTP from the tested network namespace.

### Issue 5: Apply Absolute DNS Hostname

After investigating the DNS behavior, the Loki hostname in the Fluent Bit output configuration was updated.

**Previous Configuration:**

```ini
Host loki.logging.svc.cluster.local
```

**Updated Configuration:**

```ini
Host loki.logging.svc.cluster.local.
```

The trailing dot makes the hostname an absolute DNS name.

This was done to avoid the observed DNS search-domain expansion behavior.

### Step 6: Upgrade Fluent Bit

After updating the configuration, Fluent Bit was upgraded:

```bash
helm upgrade fluent-bit fluent/fluent-bit \
  --namespace logging \
  --version 0.58.3 \
  -f fluent-bit-values.yaml
```

### Step 7: Verify Fluent Bit Logs Again

The following command was used to check recent DNS and output errors:

```bash
kubectl logs -n logging \
  -l app.kubernetes.io/name=fluent-bit \
  --since=2m --prefix |
  grep -E 'getaddrinfo|upstream|error|failed to flush'
```

**Observed Result:**

No matching output was returned during the verification period.

This indicated that the previously observed DNS and output errors were no longer appearing in the checked recent Fluent Bit logs.

**Troubleshooting Conclusion:**

- Loki service was available.
- CoreDNS pods and endpoints were available.
- DNS resolution was tested from different worker nodes.
- DNS resolution from the Fluent Bit pod's network namespace was successful during the debug test.
- Loki readiness endpoint was reachable over HTTP.
- The Loki hostname was updated to use an absolute DNS name with a trailing dot.
- After the Helm upgrade, the recent Fluent Bit logs no longer showed the matching DNS/output errors.
- Successful log ingestion was subsequently verified through the Loki API.

**Note:** The trailing-dot change correlated with successful operation. The exact underlying cause of the initial DNS failure was not conclusively established.

## 13.13 Verify Log Ingestion in Loki

After resolving the observed forwarding issue, log ingestion was verified directly through Loki's HTTP API.

### Step 1: Port-Forward Loki

Execute:

```bash
kubectl port-forward -n logging svc/loki 3100:3100
```

Keep this terminal running.

Loki can now be accessed locally at:

```text
http://localhost:3100
```

### Step 2: Query Kubernetes Container Logs

Execute the following query from another terminal:

```bash
curl -G -s "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="fluent-bit"}' \
  --data-urlencode 'limit=5' \
  --data-urlencode "start=$(date -u -d '5 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date -u +%s)000000000"
```

**Query Explanation:**

| Parameter | Purpose |
|---|---|
| `/loki/api/v1/query_range` | Loki endpoint for querying logs over a time range. |
| `job="fluent-bit"` | Filters streams having the configured Fluent Bit job label. |
| `limit=5` | Limits the returned log entries. |
| `start` | Defines the beginning of the query time range. |
| `end` | Defines the end of the query time range. |

**Observed Result:**

Loki returned:

```json
{
  "status": "success",
  "data": {
    "resultType": "streams",
    "result": []
  }
}
```

The actual response contained log streams and entries from Kubernetes containers.

Examples of observed sources included:

- kindnet-cni.
- kube-state-metrics.

The returned log streams contained Kubernetes metadata labels.

### Verified Labels

```text
job=fluent-bit
kubernetes_container_name
kubernetes_namespace_name
kubernetes_pod_name
```

This confirmed that Kubernetes container logs were successfully reaching Loki with the expected metadata.

## 13.14 Verify Fluent Bit's Own Logs in Loki

Apart from Kubernetes workload logs, Fluent Bit's own container logs were also checked in Loki.

This verification helped confirm that Fluent Bit was collecting and forwarding its own container logs as well.

Execute:

```bash
curl -G -s "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="fluent-bit",kubernetes_container_name="fluent-bit"}' \
  --data-urlencode 'limit=5' \
  --data-urlencode "start=$(date -u -d '10 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date -u +%s)000000000"
```

### Observed Result

Loki returned successful query results containing Fluent Bit container log entries.

An example of an observed log message was:

```text
[input:tail:tail.0] inotify_fs_add()
```

The returned stream included:

```text
kubernetes_container_name=fluent-bit
kubernetes_namespace_name=logging
kubernetes_pod_name=fluent-bit-...
```

This verified that Fluent Bit's own container logs were also available in Loki.

## 13.15 Important Query Note

During verification, the following endpoint was initially tried:

```text
/loki/api/v1/query
```

The API returned an error indicating that log queries are not supported as an instant query type.

For log retrieval over a time range, the correct endpoint used in this POC was:

```text
/loki/api/v1/query_range
```

This endpoint accepts a start and end timestamp and returns log streams within the requested time range.

## 13.16 Final Deployment Result

| Component / Verification | Expected Result | Observed Result |
|---|---|---|
| Helm Release | Deployed | Deployed |
| Fluent Bit DaemonSet | 3/3 Ready | 3/3 Ready |
| Fluent Bit Pods | 3 Running | 3 Running |
| Control-Plane Scheduling | Allowed | Successful |
| Worker Node 1 Scheduling | Allowed | Successful |
| Worker Node 2 Scheduling | Allowed | Successful |
| Loki Service | Accessible | Accessible |
| DNS Troubleshooting | No recurring matching errors in recent logs | Verified |
| Loki Readiness Test | `ready` | `ready` |
| Kubernetes Log Ingestion | Successful | Verified |
| Fluent Bit Self-Log Ingestion | Successful | Verified |
| Kubernetes Metadata | Attached | Verified |
| Loki Query API | Successful | Verified |

## 13.17 Current Log Collection Scope and Limitations

The current POC focuses on Kubernetes container logs.

The following points define the current implementation scope:

1. Fluent Bit reads container logs from `/var/log/containers/*.log`.
2. Kubernetes metadata is attached through the Kubernetes filter.
3. The configured Loki output matches `kube.*` records.
4. Systemd input configuration may exist in the chart defaults, but the current setup does not mount `/run/log/journal` for systemd journal collection.
5. The Loki output matches `kube.*`, so systemd records tagged `host.*` are not forwarded through this output.
6. Log ingestion and querying have been verified, but production-scale throughput, resource utilization, buffering behavior, and long-term retention have not yet been benchmarked.
7. The current Loki deployment uses SingleBinary with one replica, so the overall logging pipeline is not yet highly available.

## 13.18 Useful Troubleshooting and Verification Commands

| Task | Command |
|---|---|
| Check Fluent Bit pods | `kubectl get pods -n logging -l app.kubernetes.io/name=fluent-bit -o wide` |
| Check Fluent Bit DaemonSet | `kubectl get ds -n logging` |
| Check Fluent Bit logs | `kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --prefix` |
| Check recent DNS/output errors | `kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --since=2m --prefix` |
| Check Helm release | `helm list -n logging` |
| Check applied Helm values | `helm get values fluent-bit -n logging -o yaml` |
| Check Loki service | `kubectl get svc loki -n logging` |
| Check Loki endpoints | `kubectl get endpoints loki -n logging` |
| Check CoreDNS pods | `kubectl get pods -n kube-system` |
| Check CoreDNS logs | `kubectl logs -n kube-system -l k8s-app=kube-dns --prefix` |
| Port-forward Loki | `kubectl port-forward -n logging svc/loki 3100:3100` |
| Check Loki readiness | `curl http://localhost:3100/ready` |

## 13.19 Final Architecture

The final log collection architecture implemented in this POC is:

```text
                  Kind Kubernetes Cluster
                     ha-observability
                             |
             +---------------+---------------+
             |               |               |
       Control Plane      Worker 1        Worker 2
             |               |               |
        Container Logs  Container Logs  Container Logs
             |               |               |
        Fluent Bit      Fluent Bit      Fluent Bit
             |               |               |
             +---------------+---------------+
                             |
                    Kubernetes Metadata
                        Enrichment
                             |
                        Loki Output
                             |
                             v
                    Loki Service :3100
                             |
                             v
                       Loki Storage
                             |
                             v
                     Loki Query API
                             |
                             v
                          Grafana
```

# 14. Tempo Setup

## 14.1 Overview

Grafana Tempo is an open-source distributed tracing backend used to collect, store, and query traces generated by applications and services.

In this POC, Tempo is deployed in the `tracing` namespace of the `ha-observability` Kind Kubernetes cluster.

Tempo is responsible for storing distributed traces, while the OpenTelemetry Collector is planned to receive and forward telemetry data to Tempo.

The objective of this setup is to:

- Deploy Grafana Tempo using Helm.
- Configure Tempo in Single Binary mode.
- Configure persistent storage for trace data.
- Expose OTLP endpoints for receiving traces.
- Verify Tempo pod health and readiness.
- Send a test trace through the OTLP HTTP endpoint.
- Retrieve the same trace using the Tempo Query API.

**Current Status:**

- Tempo deployed successfully using Helm.
- Single Binary deployment mode configured.
- One Tempo replica running in the `tracing` namespace.
- 5Gi PersistentVolumeClaim created and Bound.
- Local filesystem storage configured.
- Tempo readiness endpoint verified successfully.
- OTLP HTTP endpoint verified.
- Test trace successfully ingested.
- Same test trace successfully retrieved through the Query API.
- OpenTelemetry Collector integration with Tempo is pending.
- Production HA and failure testing are pending.

## 14.2 Why We Selected This Approach

### 1. Single Binary Mode

Tempo supports different deployment modes, including Single Binary and distributed deployment.

For this POC, Single Binary mode was selected.

Single Binary mode runs the required Tempo components together in one pod instead of deploying them as separate distributed services.

**Why?**

- Simplifies the initial deployment.
- Requires fewer Kubernetes resources.
- Makes troubleshooting easier.
- Suitable for validating basic trace ingestion and retrieval.

### 2. One Replica

Tempo is configured with one replica.

```yaml
replicas: 1
```

**Why?**

One replica is sufficient for validating Tempo installation, trace ingestion, storage, and querying in the current local POC.

**Important HA Limitation:** A single Tempo replica does not provide high availability. If the pod becomes unavailable, trace ingestion and querying may be interrupted until it recovers.

### 3. Local Filesystem Storage

For this POC, Tempo uses local filesystem storage.

```yaml
storage:
  trace:
    backend: local
```

**Why?**

- Keeps the initial setup simple.
- Avoids introducing an external object storage dependency.
- Helps validate basic trace storage and retrieval.

S3-compatible object storage or MinIO can be evaluated in a future phase.

### 4. Persistent Storage

A 5Gi PersistentVolumeClaim is configured for Tempo.

**Why?**

Persistent storage allows Tempo to store trace data outside the container's writable filesystem.

The current setup uses the Kind `standard` StorageClass backed by local-path storage.

### 5. Resource and Probe Configuration

During the initial deployment, Tempo experienced readiness and liveness probe failures.

Resource allocation and probe timings were increased to provide more time for Tempo to start and respond to health checks.

The updated configuration resulted in a healthy Tempo pod.

### 6. Helm-Based Deployment

Helm is used to install and manage Tempo.

It provides a centralized configuration file through which replica count, storage, resources, and probe settings can be maintained.

## 14.3 Deployment Details

| Configuration | Value |
|---|---|
| Kubernetes Cluster | `ha-observability` |
| Namespace | `tracing` |
| Helm Release | `tempo` |
| Helm Repository | `grafana` |
| Helm Chart | `grafana/tempo` |
| Chart Version | `1.24.4` |
| Tempo Application Version | `2.9.0` |
| Deployment Mode | Single Binary |
| Resource Type | StatefulSet |
| Replicas | 1 |
| Storage Backend | Local Filesystem |
| Persistent Storage | 5Gi |
| StorageClass | `standard` |
| Query API Port | `3200` |
| OTLP gRPC Port | `4317` |
| OTLP HTTP Port | `4318` |

## 14.4 Helm Repository Setup

The Grafana Helm repository was already added during the Loki installation.

Verify the repository:

```bash
helm repo list
```

Update the repository:

```bash
helm repo update grafana
```

Check available Tempo chart versions:

```bash
helm search repo grafana/tempo --versions
```

For this POC, chart version `1.24.4` was selected.

Download the default Helm values:

```bash
helm show values grafana/tempo \
  --version 1.24.4 > tempo-default-values.yaml
```

This file contains the default configuration provided by the Tempo Helm chart.

## 14.5 Directory Structure

Tempo configuration files are maintained separately inside the project directory.

```text
ha-observability-poc/
│
├── kind/
│   └── kind-config.yaml
│
├── node-exporter/
│   └── node-exporter.yaml
│
├── kube-state-metrics/
│   └── kube-state-metrics.yaml
│
├── victoria-metrics/
│   └── values.yaml
│
├── vmagent/
│   └── vmagent-default-values.yaml
│
├── loki/
│   └── loki-values.yaml
│
├── fluent-bit/
│   └── fluent-bit-values.yaml
│
├── tempo/
│   ├── tempo-default-values.yaml
│   ├── tempo-values.yaml
│   └── tempo-rendered.yaml
│
└── README.md
```

**File Description:**

| File | Purpose |
|---|---|
| `tempo-default-values.yaml` | Default values provided by the Tempo Helm chart. |
| `tempo-values.yaml` | Custom configuration used for Tempo deployment. |
| `tempo-rendered.yaml` | Kubernetes manifests generated from the Helm chart and custom values. |

## 14.6 Tempo Configuration

### Configuration File

**Directory Path:**

`~/ha-observability-poc/tempo`

Create the directory:

```bash
mkdir -p ~/ha-observability-poc/tempo
cd ~/ha-observability-poc/tempo
```

Create the custom values file:

```bash
nano tempo-values.yaml
```

**File Path:**

`tempo/tempo-values.yaml`

### Configuration Used in the POC

```yaml
replicas: 1

tempo:
  resources:
    requests:
      cpu: 500m
      memory: 1536Mi
    limits:
      cpu: "2"
      memory: 3Gi

  livenessProbe:
    httpGet:
      path: /ready
      port: 3200
    initialDelaySeconds: 90
    periodSeconds: 10
    timeoutSeconds: 10
    failureThreshold: 6

  readinessProbe:
    httpGet:
      path: /ready
      port: 3200
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 10
    failureThreshold: 6

  storage:
    trace:
      backend: local
      local:
        path: /var/tempo/traces
      wal:
        path: /var/tempo/wal

persistence:
  enabled: true
  accessModes:
    - ReadWriteOnce
  size: 5Gi
```

## 14.7 Configuration Explanation

### 1. Replicas

```yaml
replicas: 1
```

Only one Tempo replica is deployed in the Kubernetes cluster.

This is sufficient for the initial POC but does not provide high availability.

### 2. Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: 500m
    memory: 1536Mi
  limits:
    cpu: "2"
    memory: 3Gi
```

Resource requests define the minimum CPU and memory requested by the Tempo container.

Resource limits define the maximum CPU and memory that the container can consume.

These values were increased after the initial deployment experienced health-check failures.

### 3. Liveness Probe

```yaml
livenessProbe:
  httpGet:
    path: /ready
    port: 3200
  initialDelaySeconds: 90
  periodSeconds: 10
  timeoutSeconds: 10
  failureThreshold: 6
```

The liveness probe checks whether the Tempo process is responding to its health endpoint.

If the liveness probe repeatedly fails, Kubernetes can restart the container.

### 4. Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 3200
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 10
  failureThreshold: 6
```

The readiness probe checks whether Tempo is ready to receive traffic.

If the readiness probe fails, Kubernetes does not consider the pod ready to serve traffic.

### 5. Trace Storage Backend

```yaml
storage:
  trace:
    backend: local
```

Tempo uses local filesystem storage for trace data.

The trace data directory is:

```text
/var/tempo/traces
```

The Write-Ahead Log (WAL) directory is:

```text
/var/tempo/wal
```

### 6. Persistent Volume Claim

```yaml
persistence:
  enabled: true
  accessModes:
    - ReadWriteOnce
  size: 5Gi
```

Persistent storage is enabled with a capacity of 5Gi.

The PVC is used to provide persistent storage to the Tempo pod.

The current `standard` StorageClass uses the local-path provisioner.

## 14.8 Render and Validate Helm Templates

Before installing Tempo, render the Helm templates.

```bash
helm template tempo grafana/tempo \
  --namespace tracing \
  --version 1.24.4 \
  -f tempo-values.yaml \
  > tempo-rendered.yaml
```

Verify the StatefulSet:

```bash
grep -n -A 45 'kind: StatefulSet' tempo-rendered.yaml
```

Verify the PersistentVolumeClaim configuration:

```bash
grep -n -A 20 'volumeClaimTemplates:' tempo-rendered.yaml
```

Verify the resource configuration:

```bash
grep -n -A 15 'resources:' tempo-rendered.yaml
```

Verify the probe configuration:

```bash
grep -n -A 20 'livenessProbe:' tempo-rendered.yaml
```

The rendered manifest can be inspected before applying the configuration to the Kubernetes cluster.

## 14.9 Install Tempo

Create the tracing namespace if it does not already exist:

```bash
kubectl create namespace tracing \
  --dry-run=client -o yaml | kubectl apply -f -
```

Install Tempo using Helm:

```bash
helm upgrade --install tempo grafana/tempo \
  --namespace tracing \
  --version 1.24.4 \
  -f tempo-values.yaml
```

Check the Helm release:

```bash
helm list -n tracing
```

Check the release status:

```bash
helm status tempo -n tracing
```

## 14.10 Verify Tempo Deployment

### Check Tempo Pod

```bash
kubectl get pods -n tracing -o wide
```

**Observed Deployment Result:**

| Pod | Ready | Status | Restarts |
|---|---|---|---|
| `tempo-0` | 1/1 | Running | 0 |

The Tempo pod was observed running on:

`ha-observability-worker2`

### Check Tempo StatefulSet

```bash
kubectl get statefulset -n tracing
```

### Check Tempo Logs

```bash
kubectl logs tempo-0 -n tracing
```

These commands help verify the pod status and inspect application logs during startup or troubleshooting.

## 14.11 Verify PersistentVolumeClaim

Check the PVC:

```bash
kubectl get pvc -n tracing
```

**Observed Result:**

| PVC | Capacity | Status | Access Mode |
|---|---|---|---|
| `storage-tempo-0` | 5Gi | Bound | RWO |

Check the StorageClass:

```bash
kubectl get storageclass standard
```

The current POC uses the `standard` StorageClass backed by `rancher.io/local-path`.

The PVC was successfully created and reached the `Bound` state.

**Storage Limitation:** The current local-path storage is not shared multi-node storage. It should not be treated as a production distributed storage solution.

## 14.12 Verify Tempo Services

Check the services:

```bash
kubectl get svc -n tracing
```

The Tempo Service exposes the endpoints required for trace ingestion and querying.

### Important Tempo Ports

| Port | Purpose |
|---|---|
| `3200` | Tempo HTTP Query API and readiness endpoint |
| `4317` | OTLP gRPC trace ingestion |
| `4318` | OTLP HTTP trace ingestion |
| `14250` | Jaeger gRPC |
| `14268` | Jaeger HTTP |
| `9411` | Zipkin HTTP |

The current POC specifically verifies:

- OTLP HTTP ingestion on port `4318`.
- Trace retrieval through the Query API on port `3200`.

## 14.13 Tempo Readiness Endpoint Verification

Tempo exposes a readiness endpoint on port `3200`.

### Step 1: Port-Forward Tempo Service

```bash
kubectl port-forward -n tracing svc/tempo 3200:3200
```

Keep this terminal running.

### Step 2: Check Readiness

Open another terminal and execute:

```bash
curl -i http://127.0.0.1:3200/ready
```

**Observed Response:**

```text
HTTP/1.1 200 OK

ready
```

This confirms that the Tempo readiness endpoint is responding successfully.

## 14.14 OTLP HTTP Trace Ingestion Testing

After verifying Tempo readiness, a manually generated test trace was sent to Tempo through its OTLP HTTP endpoint.

The purpose of this test was to verify whether Tempo could successfully receive and process a trace.

### Step 1: Port-Forward OTLP HTTP Port

```bash
kubectl port-forward -n tracing svc/tempo 4318:4318
```

Keep this terminal running.

### Step 2: Generate a Test Trace

A test trace was generated with the following details:

| Parameter | Value |
|---|---|
| Service Name | `tempo-poc` |
| Span Name | `tempo-test-span` |
| Trace ID | `5b8aa5a2d2c872e8321cf37308d69df2` |
| Span ID | `051581bf3cb55c13` |
| Span Kind | `SPAN_KIND_INTERNAL` |
| OTLP Endpoint | `http://127.0.0.1:4318/v1/traces` |

The trace was sent using the OTLP HTTP protocol.

**Endpoint:**

```text
POST http://127.0.0.1:4318/v1/traces
```

**Observed Result:**

```text
HTTP 200 OK
```

This confirmed that Tempo successfully accepted the test trace.

## 14.15 Trace Retrieval Verification

After successful ingestion, the same trace was retrieved using the Tempo Query API.

### Step 1: Port-Forward Tempo Query API

```bash
kubectl port-forward -n tracing svc/tempo 3200:3200
```

### Step 2: Retrieve the Trace

Execute:

```bash
curl -s -i \
  http://127.0.0.1:3200/api/traces/5b8aa5a2d2c872e8321cf37308d69df2
```

### Observed Response

```text
HTTP/1.1 200 OK
Content-Type: application/json
```

The response contained the following trace information:

```json
{
  "batches": [
    {
      "resource": {
        "attributes": [
          {
            "key": "service.name",
            "value": {
              "stringValue": "tempo-poc"
            }
          }
        ]
      },
      "scopeSpans": [
        {
          "scope": {},
          "spans": [
            {
              "traceId": "W4qlotLIcugyHPNzCNad8g==",
              "spanId": "BRWBvzy1XBM=",
              "name": "tempo-test-span",
              "kind": "SPAN_KIND_INTERNAL",
              "startTimeUnixNano": "1794860000000000000",
              "endTimeUnixNano": "1794860001000000000",
              "status": {}
            }
          ]
        }
      ]
    }
  ]
}
```

### Verification Explanation

The retrieved response contains the expected service and span information.

| Field | Expected Value | Observed Result |
|---|---|---|
| Service Name | `tempo-poc` | Matched |
| Span Name | `tempo-test-span` | Matched |
| Span Kind | `SPAN_KIND_INTERNAL` | Matched |
| Trace Retrieval | Requested Trace ID | Successful |
| HTTP Status | 200 OK | Successful |

The `traceId` and `spanId` values in the JSON response are Base64-encoded representations of the IDs supplied in the test trace.

**Verification Result:**

The test trace was successfully ingested through the OTLP HTTP endpoint and retrieved using the Tempo Query API.

This confirms that the basic end-to-end trace ingestion and retrieval workflow is working.

## 14.16 Troubleshooting Performed During Tempo Setup

During the initial Tempo deployment, the pod experienced readiness and liveness probe failures.

### Issue 1: Readiness and Liveness Probe Failures

**Observed Error:**

```text
Readiness probe failed:
Get "http://<pod-ip>:3200/ready":
context deadline exceeded
```

The pod was not consistently responding to health checks within the configured timeout.

### Issue 2: Container Termination

During the initial deployment, the container also showed an earlier termination with exit code `137`.

Exit code 137 indicates that the container was terminated with SIGKILL.

This exit code alone does not confirm that the termination was caused by an out-of-memory event.

### Resolution

The configuration was updated with:

- Increased CPU and memory requests.
- Increased CPU and memory limits.
- Increased liveness probe initial delay.
- Increased readiness probe initial delay.
- Increased probe timeout.
- Increased probe failure threshold.

The updated configuration was applied using Helm upgrade.

```bash
helm upgrade tempo grafana/tempo \
  --namespace tracing \
  --version 1.24.4 \
  -f tempo-values.yaml
```

### Final Verification

```bash
kubectl get pods -n tracing
```

**Observed Result:**

```text
NAME      READY   STATUS    RESTARTS
tempo-0   1/1     Running   0
```

The readiness endpoint also returned HTTP 200.

This confirmed that the updated configuration resolved the observed startup and health-check issues during the verification period.

## 14.17 Current Tempo Architecture

The following architecture represents the Tempo deployment implemented in the current POC.

```text
                  Kind Kubernetes Cluster
                     ha-observability
                           |
                           |
                    tracing Namespace
                           |
                           v
                  +------------------+
                  |   Tempo Service  |
                  +------------------+
                           |
              +------------+------------+
              |                         |
              v                         v
        OTLP HTTP :4318            Query API :3200
              |                         |
              v                         v
        +-----------------------------------+
        |             Tempo Pod             |
        |                                   |
        |          Single Binary            |
        |                                   |
        |        Trace Ingestion            |
        |        Trace Processing           |
        |        Trace Query                |
        +-----------------------------------+
                           |
                           v
                  +------------------+
                  | Persistent Volume|
                  |                  |
                  |      5Gi PVC     |
                  |                  |
                  |  Local Filesystem|
                  +------------------+
                           |
                           v
                    Trace Storage
```

### Architecture Flow Explanation

1. A test client sends trace data through the OTLP HTTP endpoint.
2. Tempo receives the trace on port `4318`.
3. Tempo processes the incoming trace.
4. Trace data is stored using the configured local filesystem backend.
5. The Query API on port `3200` is used to retrieve the trace.
6. The returned response contains the service name and span information.

**Note:** The current diagram represents the manually verified trace ingestion flow. OpenTelemetry Collector integration is planned but has not yet been verified.

## 14.18 Final Deployment Result

| Component / Verification | Expected Result | Observed Result |
|---|---|---|
| Helm Release | Deployed | Deployed |
| Tempo Deployment Mode | Single Binary | Configured |
| Tempo Replicas | 1 | 1 |
| Tempo Pod | Running and Ready | 1/1 Running |
| Restart Count After Final Upgrade | 0 | 0 |
| PersistentVolumeClaim | Bound | Bound |
| Storage Capacity | 5Gi | 5Gi |
| Storage Backend | Local Filesystem | Configured |
| Readiness Endpoint | HTTP 200 | Verified |
| OTLP HTTP Endpoint | Accessible | Verified |
| Test Trace Ingestion | Successful | Verified |
| Trace Retrieval | Successful | Verified |
| Service Name | `tempo-poc` | Matched |
| Span Name | `tempo-test-span` | Matched |

## 14.19 Current Scope and Limitations

The current Tempo implementation focuses on validating basic trace ingestion, storage, and retrieval in a local Kubernetes environment.

The following limitations apply:

1. **Single Replica:** Only one Tempo replica is deployed. The current setup does not provide high availability.

2. **Local Filesystem Storage:** Trace data is stored on local persistent storage rather than distributed object storage.

3. **Storage Dependency:** The `standard` StorageClass uses local-path storage and does not provide shared multi-node storage.

4. **OTel Collector Integration Pending:** A manually generated test trace was successfully ingested and retrieved. However, automatic trace export from the OpenTelemetry Collector to Tempo has not yet been verified.

5. **No Production HA Validation:** Pod failure, storage recovery, and production-scale failure scenarios have not yet been tested.

6. **Performance Testing Pending:** High-volume trace ingestion, query performance, and long-term storage behavior have not yet been benchmarked.

### Future Improvements

- Configure OpenTelemetry Collector to export traces to Tempo.
- Verify automatic trace ingestion from the Collector.
- Evaluate S3-compatible object storage or MinIO for trace storage.
- Evaluate distributed Tempo deployment for production workloads.
- Perform pod failure and storage recovery testing.
- Integrate Tempo as a Grafana data source and validate trace exploration.

## 14.20 Useful Troubleshooting and Verification Commands

| Task | Command |
|---|---|
| Check Tempo pods | `kubectl get pods -n tracing -o wide` |
| Check Tempo StatefulSet | `kubectl get statefulset -n tracing` |
| Check Tempo logs | `kubectl logs tempo-0 -n tracing` |
| Check Tempo services | `kubectl get svc -n tracing` |
| Check Tempo PVC | `kubectl get pvc -n tracing` |
| Check Helm release | `helm list -n tracing` |
| Check Helm values | `helm get values tempo -n tracing -o yaml` |
| Port-forward Query API | `kubectl port-forward -n tracing svc/tempo 3200:3200` |
| Check readiness | `curl -i http://127.0.0.1:3200/ready` |
| Port-forward OTLP HTTP | `kubectl port-forward -n tracing svc/tempo 4318:4318` |
| Retrieve test trace | `curl -s -i http://127.0.0.1:3200/api/traces/5b8aa5a2d2c872e8321cf37308d69df2` |

## 14.21 Final Outcome

As part of the High Availability Observability Stack POC, Grafana Tempo was successfully deployed in the `tracing` namespace using Helm.

The deployment uses Single Binary mode with one replica and a 5Gi PersistentVolumeClaim backed by local filesystem storage.

Initially, the Tempo pod experienced readiness and liveness probe failures. Resource allocation and probe timings were adjusted, after which the pod became healthy.

The readiness endpoint returned HTTP 200.

A manually generated test trace was sent through the OTLP HTTP endpoint on port `4318`. The same trace was then successfully retrieved using the Tempo Query API on port `3200`.

Both trace ingestion and retrieval returned HTTP 200, confirming that the basic trace ingestion and retrieval workflow is working successfully.

The next step is to integrate the OpenTelemetry Collector with Tempo so that traces can be exported automatically from telemetry sources instead of relying on manual test requests.

# 15. OpenTelemetry Collector Setup

## 15.1 Overview

OpenTelemetry Collector is used to receive, process, and export telemetry data such as logs, metrics, and traces.

In our Kubernetes observability POC, we deployed OpenTelemetry Collector as a DaemonSet in the `tracing` namespace.

Our main objective was to configure OpenTelemetry Collector to receive traces from applications and forward them to Grafana Tempo.

For this POC, we configured only the traces pipeline.

- Logs pipeline: Disabled
- Metrics pipeline: Disabled
- Traces pipeline: Enabled

The Collector receives traces through OTLP protocols and forwards them to Tempo using OTLP gRPC.

## 15.2 Deployment Details

| Component | Configuration |
|---|---|
| Component | OpenTelemetry Collector |
| Namespace | tracing |
| Deployment Mode | DaemonSet |
| Helm Release | otel-collector |
| Helm Repository | open-telemetry |
| Helm Chart | opentelemetry-collector |
| Chart Version | 0.178.3 |
| App Version | 0.163.2 |
| Image | otel/opentelemetry-collector-contrib |
| Collector Pods | 3 |
| OTLP gRPC | 4317 |
| OTLP HTTP | 4318 |
| Health Check | 13133 |
| Exporter | OTLP |
| Backend | Tempo |
| Logs Pipeline | Disabled |
| Metrics Pipeline | Disabled |
| Traces Pipeline | Enabled |

## 15.3 Why We Selected DaemonSet Mode

Our Kind cluster contains three Kubernetes nodes:

| Node | Role |
|---|---|
| ha-observability-control-plane | Control Plane |
| ha-observability-worker | Worker |
| ha-observability-worker2 | Worker |

We selected DaemonSet mode so that one OpenTelemetry Collector pod runs on each Kubernetes node.

This provides node-level Collector coverage. Kubernetes automatically maintains the Collector pods on eligible nodes.

## 15.4 Helm Repository Setup

First, we added the OpenTelemetry Helm repository.

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

Next, we updated the Helm repository information.

```bash
helm repo update
```

The repository was already configured in our environment, so Helm displayed:

```text
"open-telemetry" already exists with the same configuration, skipping
```

This was normal and not an error.

## 15.5 Fetching Default Helm Values

Before preparing our custom configuration, we downloaded the default values provided by the OpenTelemetry Collector Helm chart.

```bash
helm show values open-telemetry/opentelemetry-collector \
  --version 0.178.3 > default-values.yaml
```

The command downloaded the chart's default configuration and saved it in `default-values.yaml`.

We inspected this file to understand the available settings, including:

- Deployment mode
- Presets
- Receivers
- Processors
- Exporters
- Service pipelines
- Ports
- Resources
- Health probes
- Kubernetes Service configuration

### 15.5.1 Inspecting Default Configuration

We searched for the important configuration sections.

```bash
grep -nE 'receivers:|processors:|exporters:|service:|pipelines:|logs:|metrics:|traces:|otlp:' default-values.yaml
```

The output displayed the line numbers of the main configuration sections.

We then inspected the receiver, processor, exporter, and pipeline configuration.

```bash
sed -n '250,340p' default-values.yaml
```

### 15.5.2 Default Exporter

The default chart configuration contained:

```yaml
exporters:
  debug: {}
```

The Debug exporter is used to display telemetry data for debugging.

For our POC, we replaced it with an OTLP exporter connected to Tempo.

### 15.5.3 Default Extensions

The chart included the health check extension.

```yaml
extensions:
  health_check:
    endpoint: ${env:MY_POD_IP}:13133
```

This extension provides a health endpoint used by Kubernetes liveness and readiness probes.

We retained this extension in our custom configuration.

### 15.5.4 Default Processors

The default configuration included:

```yaml
processors:
  batch: {}

  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
```

- Memory Limiter: Helps control Collector memory consumption.
- Batch: Groups telemetry data before exporting it.

We retained both processors for our traces pipeline.

### 15.5.5 Default Receivers

The default chart provided multiple receivers.

```yaml
receivers:
  jaeger:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:14250
      thrift_http:
        endpoint: ${env:MY_POD_IP}:14268
      thrift_compact:
        endpoint: ${env:MY_POD_IP}:6831

  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318

  prometheus:
    config:
      scrape_configs:
        - job_name: opentelemetry-collector
          scrape_interval: 10s
          static_configs:
            - targets:
                - ${env:MY_POD_IP}:8888

  zipkin:
    endpoint: ${env:MY_POD_IP}:9411
```

The available receivers were:

| Receiver | Purpose |
|---|---|
| OTLP | Receive OpenTelemetry data |
| Jaeger | Receive Jaeger traces |
| Prometheus | Scrape Prometheus metrics |
| Zipkin | Receive Zipkin traces |

For our POC, we selected only the OTLP receiver.

## 15.6 Understanding Default Pipelines

The default Helm chart configuration contained three pipelines.

**Logs Pipeline:**

```yaml
logs:
  exporters:
    - debug
  processors:
    - memory_limiter
    - batch
  receivers:
    - otlp
```

**Metrics Pipeline:**

```yaml
metrics:
  exporters:
    - debug
  processors:
    - memory_limiter
    - batch
  receivers:
    - otlp
    - prometheus
```

**Traces Pipeline:**

```yaml
traces:
  exporters:
    - debug
  processors:
    - memory_limiter
    - batch
  receivers:
    - otlp
    - jaeger
    - zipkin
```

The default chart connects all three pipelines to the Debug exporter.

However, our requirement was to use OpenTelemetry Collector only for traces and forward them to Tempo.

Therefore, we configured a separate traces-only pipeline.

## 15.7 Inspecting alternateConfig

We inspected the `alternateConfig` section.

```bash
sed -n '340,470p' default-values.yaml
```

The default configuration contained:

```yaml
alternateConfig: {}
```

The Helm chart supports `alternateConfig` for providing a complete custom Collector configuration.

When this option is used:

- The custom configuration replaces the default Collector configuration.
- The default configuration is not merged with it.
- Required receivers, processors, exporters, extensions, and pipelines must be defined manually.

We selected `alternateConfig` to keep only the traces pipeline and avoid retaining the default logs and metrics pipelines.

## 15.8 Helm Chart Version Verification

We checked the Helm chart information.

```bash
helm show chart open-telemetry/opentelemetry-collector
```

The chart information showed:

```yaml
name: opentelemetry-collector
version: 0.178.3
appVersion: 0.163.2
```

We used chart version `0.178.3` for our deployment.

We also downloaded the default values for this exact version:

```bash
helm show values open-telemetry/opentelemetry-collector \
  --version 0.178.3 > default-values.yaml
```

## 15.9 Inspecting Presets

We inspected the available presets in the default values file.

```bash
sed -n '1,160p' default-values.yaml
```

The chart provides presets for collecting logs, host metrics, Kubernetes metadata, kubelet metrics, Kubernetes events, and other telemetry.

For our POC, these collection presets were kept disabled.

| Preset | Configuration |
|---|---|
| logsCollection | Disabled |
| hostMetrics | Disabled |
| kubernetesAttributes | Disabled |
| kubeletMetrics | Disabled |
| kubernetesEvents | Disabled |
| kubernetesObjects | Disabled |
| clusterMetrics | Disabled |
| annotationDiscovery | Disabled |
| profiling | Disabled |
| resourceDetection | Disabled |

This keeps our configuration focused on trace ingestion and forwarding.

## 15.10 Inspecting Resources and Deployment Settings

We checked the default resource configuration.

```bash
grep -n -A20 '^resources:' default-values.yaml
```

The default values showed:

```yaml
resources: {}
```

No custom CPU or memory requests and limits were configured in the default values.

For this initial POC, we retained the chart's default resource settings.

## 15.11 Inspecting Ports and Service Configuration

We inspected the available ports.

```bash
grep -n -A45 '^ports:' default-values.yaml
```

The chart provides the following ports:

| Port | Purpose |
|---|---|
| 4317 | OTLP gRPC |
| 4318 | OTLP HTTP |
| 6831 | Jaeger Compact UDP |
| 14268 | Jaeger Thrift HTTP |
| 14250 | Jaeger gRPC |
| 9411 | Zipkin |
| 8888 | Collector Prometheus metrics |

For our POC, we enabled only OTLP gRPC and OTLP HTTP.

We also inspected the Service configuration.

```bash
grep -n -A20 '^service:' default-values.yaml
```

The chart requires explicit Service enablement when running in DaemonSet mode.

We configured a ClusterIP Service to expose the OTLP ports inside the Kubernetes cluster.

## 15.12 Inspecting Health Probes

We inspected the default liveness, readiness, and startup probe configuration.

```bash
sed -n '600,690p' default-values.yaml
```

The default configuration contained:

```yaml
livenessProbe:
  httpGet:
    port: 13133
    path: /

readinessProbe:
  httpGet:
    port: 13133
    path: /

startupProbe: {}
```

We retained the default liveness and readiness probes.

The health check extension listens on port `13133`.

## 15.13 Creating Custom otel-values.yaml

After inspecting the default Helm configuration, we created our custom values file.

```bash
nano otel-values.yaml
```

### Complete otel-values.yaml

```yaml
mode: daemonset

image:
  repository: otel/opentelemetry-collector-contrib

alternateConfig:
  extensions:
    health_check:
      endpoint: 0.0.0.0:13133

  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

  processors:
    memory_limiter:
      check_interval: 5s
      limit_percentage: 80
      spike_limit_percentage: 25
    batch: {}

  exporters:
    otlp:
      endpoint: tempo.tracing.svc.cluster.local:4317
      tls:
        insecure: true

  service:
    extensions:
      - health_check
    pipelines:
      traces:
        receivers:
          - otlp
        processors:
          - memory_limiter
          - batch
        exporters:
          - otlp

presets:
  logsCollection:
    enabled: false
  hostMetrics:
    enabled: false
  kubernetesAttributes:
    enabled: false
  kubeletMetrics:
    enabled: false
  kubernetesEvents:
    enabled: false
  kubernetesObjects:
    enabled: false
  clusterMetrics:
    enabled: false
  annotationDiscovery:
    logs:
      enabled: false
    metrics:
      enabled: false
  profiling:
    enabled: false
  resourceDetection:
    enabled: false

service:
  enabled: true
  type: ClusterIP

ports:
  otlp:
    enabled: true
  otlp-http:
    enabled: true
  jaeger-compact:
    enabled: false
  jaeger-thrift:
    enabled: false
  jaeger-grpc:
    enabled: false
  zipkin:
    enabled: false
  metrics:
    enabled: false

tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

## 15.14 Configuration Explanation

### 1. DaemonSet Mode

```yaml
mode: daemonset
```

Runs one Collector pod on every eligible Kubernetes node.

### 2. OTLP Receiver

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```

Accepts trace data through OTLP gRPC and HTTP.

### 3. Processors

```yaml
processors:
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
  batch: {}
```

Memory Limiter helps control memory usage, while Batch groups traces before exporting them.

### 4. OTLP Exporter

```yaml
exporters:
  otlp:
    endpoint: tempo.tracing.svc.cluster.local:4317
    tls:
      insecure: true
```

Forwards traces to the Tempo Service using OTLP gRPC.

### 5. Traces Pipeline

```yaml
service:
  extensions:
    - health_check
  pipelines:
    traces:
      receivers:
        - otlp
      processors:
        - memory_limiter
        - batch
      exporters:
        - otlp
```

This pipeline receives traces, processes them, and forwards them to Tempo.

Only the traces pipeline is defined.

### 6. Service Configuration

```yaml
service:
  enabled: true
  type: ClusterIP
```

Creates an internal Kubernetes Service for the Collector.

### 7. Port Configuration

Only OTLP ports are enabled.

```yaml
ports:
  otlp:
    enabled: true
  otlp-http:
    enabled: true
  jaeger-compact:
    enabled: false
  jaeger-thrift:
    enabled: false
  jaeger-grpc:
    enabled: false
  zipkin:
    enabled: false
  metrics:
    enabled: false
```

### 8. Control Plane Toleration

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

Our control-plane node has a `NoSchedule` taint.

This toleration allows the Collector DaemonSet to schedule a pod on the control-plane node as well.

## 15.15 Helm Template Validation

Before installing the Collector, we generated the Kubernetes manifests.

```bash
helm template otel-collector open-telemetry/opentelemetry-collector \
  --version 0.178.3 \
  --namespace tracing \
  -f otel-values.yaml > otel-rendered.yaml
```

The rendered manifests were saved in `otel-rendered.yaml`.

### Verify DaemonSet and Service

```bash
grep -nE 'kind: DaemonSet|kind: Service|kind: Deployment' otel-rendered.yaml
```

The output confirmed:

```text
kind: ServiceAccount
kind: Service
kind: DaemonSet
```

### Verify Pipelines

```bash
grep -nE 'logs:|metrics:|traces:|tempo.tracing|health_check:' otel-rendered.yaml
```

The rendered configuration contained:

```text
endpoint: tempo.tracing.svc.cluster.local:4317
health_check:
traces:
```

Only the traces pipeline was present.

### Verify Service Ports

```bash
sed -n '65,155p' otel-rendered.yaml
```

The Service exposed:

```yaml
ports:
  - name: otlp
    port: 4317
    targetPort: 4317
    protocol: TCP

  - name: otlp-http
    port: 4318
    targetPort: 4318
    protocol: TCP
```

### Verify DaemonSet Ports

```bash
sed -n '145,230p' otel-rendered.yaml
```

The Collector container exposed OTLP ports `4317` and `4318`.

Both ports also had `hostPort` configured by the chart defaults.

### Verify Tolerations

```bash
grep -n -A5 'tolerations:' otel-rendered.yaml
```

The rendered DaemonSet contained the control-plane toleration.

## 15.16 Installing OpenTelemetry Collector

After validating the rendered Kubernetes manifests, we installed the Collector.

```bash
helm install otel-collector open-telemetry/opentelemetry-collector \
  --version 0.178.3 \
  --namespace tracing \
  -f otel-values.yaml
```

### Verify Collector Pods

```bash
kubectl get pods -o wide -n tracing
```

Observed output:

```text
NAME                                                 READY   STATUS    RESTARTS   NODE
otel-collector-opentelemetry-collector-agent-pg574   1/1     Running   0          ha-observability-control-plane
otel-collector-opentelemetry-collector-agent-qgw8r   1/1     Running   0          ha-observability-worker
otel-collector-opentelemetry-collector-agent-w6k4t   1/1     Running   0          ha-observability-worker2
tempo-0                                              1/1     Running   0          ha-observability-worker2
```

All three Collector pods were running on separate Kubernetes nodes.

## 15.17 Verifying Collector Logs

We checked the Collector logs.

```bash
kubectl logs -n tracing \
  otel-collector-opentelemetry-collector-agent-pg574 \
  --tail=50
```

The logs showed:

```text
Starting GRPC server
endpoint: [::]:4317

Starting HTTP server
endpoint: [::]:4318

Health Check state change
status: ready

Everything is ready. Begin running and processing data.
```

This confirmed that the Collector started successfully and both OTLP receivers were listening.

## 15.18 Verifying Collector Service

Command:

```bash
kubectl get svc -n tracing
```

Observed Collector Service:

```text
NAME                                     TYPE        CLUSTER-IP
otel-collector-opentelemetry-collector   ClusterIP   10.96.66.137
```

The Service exposed:

```text
4317/TCP
4318/TCP
```

Applications inside the cluster can use the following endpoints.

**OTLP gRPC:**

```text
otel-collector-opentelemetry-collector.tracing.svc.cluster.local:4317
```

**OTLP HTTP:**

```text
http://otel-collector-opentelemetry-collector.tracing.svc.cluster.local:4318
```

## 15.19 End-to-End Trace Testing

We tested trace ingestion by sending a manually generated OTLP trace through the Collector.

### Step 1: Port Forward Collector

```bash
kubectl port-forward -n tracing \
  svc/otel-collector-opentelemetry-collector 4318:4318
```

### Step 2: Send Test Trace

From another terminal, we sent a test trace using curl.

```bash
curl -i http://127.0.0.1:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": {"stringValue": "otel-collector-poc"}
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "5b8aa5a2d2c872e8321cf37308d69df3",
          "spanId": "051581bf3cb55c13",
          "name": "collector-to-tempo-test",
          "kind": 1,
          "startTimeUnixNano": "1794860000000000000",
          "endTimeUnixNano": "1794860001000000000"
        }]
      }]
    }]
  }'
```

### Step 3: Collector Response

Observed response:

```text
HTTP/1.1 200 OK

{"partialSuccess":{}}
```

The Collector accepted the trace request successfully.

## 15.20 Verifying Trace in Tempo

We then checked whether the trace was available in Tempo.

### Step 1: Port Forward Tempo

```bash
kubectl port-forward -n tracing svc/tempo 3200:3200
```

### Step 2: Retrieve Trace

```bash
curl -s http://127.0.0.1:3200/api/traces/5b8aa5a2d2c872e8321cf37308d69df3
```

Observed response:

```json
{
  "batches": [
    {
      "resource": {
        "attributes": [
          {
            "key": "service.name",
            "value": {
              "stringValue": "otel-collector-poc"
            }
          }
        ]
      },
      "scopeSpans": [
        {
          "scope": {},
          "spans": [
            {
              "name": "collector-to-tempo-test",
              "kind": "SPAN_KIND_INTERNAL"
            }
          ]
        }
      ]
    }
  ]
}
```

The response contained the expected service name and span name.

### Trace Verification Result

| Attribute | Value |
|---|---|
| Service Name | otel-collector-poc |
| Span Name | collector-to-tempo-test |
| Trace ID | 5b8aa5a2d2c872e8321cf37308d69df3 |
| Span ID | 051581bf3cb55c13 |
| Collector Response | HTTP 200 |
| Tempo Retrieval | Successful |

Tempo returned the trace successfully.

The response represents trace and span IDs in Base64 encoding, corresponding to the hexadecimal IDs sent in the original request.

This confirmed that the Collector successfully forwarded the trace to Tempo.

## 15.21 Final Architecture

```text
                 Kubernetes Cluster
                 Namespace: tracing

   Node 1              Node 2              Node 3
+------------+      +------------+      +------------+
| Control    |      | Worker     |      | Worker2    |
| Plane      |      |            |      |            |
|            |      |            |      |            |
| OTel       |      | OTel       |      | OTel       |
| Collector  |      | Collector  |      | Collector  |
| Pod        |      | Pod        |      | Pod        |
+------------+      +------------+      +------------+
       |                  |                   |
       +------------------+-------------------+
                          |
                    OTLP Receiver
                     4317 / 4318
                          |
                    Traces Pipeline
                          |
                   Memory Limiter
                          |
                        Batch
                          |
                    OTLP Exporter
                          |
                          v
              tempo.tracing.svc.cluster.local
                          |
                    OTLP gRPC 4317
                          |
                          v
                     +---------+
                     |  Tempo  |
                     |         |
                     |  Trace  |
                     | Storage |
                     +---------+
                          |
                          v
                   Query API 3200
```

## 15.22 Useful Commands

**Check Collector pods:**

```bash
kubectl get pods -n tracing -o wide
```

**Check DaemonSet:**

```bash
kubectl get daemonset -n tracing
```

**Check Services:**

```bash
kubectl get svc -n tracing
```

**Check Collector logs:**

```bash
kubectl logs -n tracing \
  -l app.kubernetes.io/instance=otel-collector \
  --tail=100
```

**Describe Collector pod:**

```bash
kubectl describe pod -n tracing \
  otel-collector-opentelemetry-collector-agent-pg574
```

**Check Helm release:**

```bash
helm list -n tracing
```

**Check installed Helm values:**

```bash
helm get values otel-collector -n tracing
```

**Check Collector ConfigMap:**

```bash
kubectl get configmap -n tracing \
  otel-collector-opentelemetry-collector-agent -o yaml
```

## 15.23 Final Deployment Result

| Component | Final Status |
|---|---|
| Helm Repository | Configured |
| Helm Chart | 0.178.3 |
| Collector App Version | 0.163.2 |
| Deployment Mode | DaemonSet |
| Collector Pods | 3/3 Running |
| OTLP gRPC Receiver | Working |
| OTLP HTTP Receiver | Working |
| Health Check | Ready |
| Logs Pipeline | Disabled |
| Metrics Pipeline | Disabled |
| Traces Pipeline | Enabled |
| Tempo Exporter | Configured |
| Test Trace Ingestion | Successful |
| Tempo Trace Retrieval | Successful |

# 16. Grafana Setup

## 16.1 Overview

Grafana is an open-source visualization and monitoring platform used to create dashboards and visualize metrics, logs, and distributed traces.

In this POC, Grafana is deployed in the `monitoring` namespace of the `ha-observability` Kubernetes cluster.

Grafana acts as the central visualization layer for the observability stack.

The existing observability components include:

- VictoriaMetrics for metrics storage.
- Loki for log storage.
- Tempo for distributed trace storage.
- OpenTelemetry Collector for receiving and forwarding telemetry data.

Grafana will be used to connect with these backends and visualize their data through dashboards.

**Current Scope:**

- Grafana is deployed with one replica.
- Persistent storage is enabled through a 5Gi PVC.
- Grafana is exposed internally through a ClusterIP Service.
- No datasource has been configured yet.
- Browser access is performed through Kubernetes port-forwarding.
- Multi-replica Grafana HA is kept for a future phase.

## 16.2 Why We Selected a Single Grafana Replica

Initially, we evaluated running Grafana with two replicas.

However, running multiple Grafana replicas requires proper database and storage architecture.

### 1. Initial Two-Replica Approach

We initially considered the following configuration:

```yaml
replicas: 2
```

The objective was to run two Grafana pods and evaluate whether both replicas could provide Grafana availability.

During the Helm template validation, we observed that the chart generated:

- One Grafana Deployment.
- Two Grafana replicas.
- One PersistentVolumeClaim named `grafana`.
- Both replicas referenced the same PVC.

The rendered configuration contained:

```yaml
replicas: 2
```

And the volume reference was:

```yaml
persistentVolumeClaim:
  claimName: grafana
```

This configuration raised a storage and database-sharing concern.

### 2. Why One PVC Is Not Suitable for Two Independent Grafana Replicas

Grafana uses SQLite by default when an external database is not configured.

SQLite stores Grafana application data in its local data directory.

This data includes:

- Dashboards created through Grafana UI.
- Datasource configurations.
- User information.
- Dashboard settings.
- Other Grafana application metadata.

If two Grafana replicas use the same SQLite database concurrently, it can lead to database locking and unsupported shared-database behavior.

The configured PVC also uses `ReadWriteOnce` access mode.

ReadWriteOnce generally allows a volume to be mounted read-write by a single node at a time. It does not provide a shared multi-node database architecture.

Therefore, simply increasing the replica count to two does not create a proper Grafana HA setup.

### 3. Why We Selected One Replica

For the current POC phase, we selected one Grafana replica because:

- The primary objective is to deploy Grafana and verify its functionality.
- We want to validate persistent storage.
- We want to test whether Grafana data survives pod recreation.
- We do not want to introduce PostgreSQL and additional database configuration at this stage.
- One replica with one PVC provides a simpler and more predictable initial deployment.

### 4. Why We Did Not Select Multi-Replica Deployment

A proper multi-replica Grafana architecture requires a shared database backend.

For example:

- Two or more Grafana replicas.
- Shared PostgreSQL database.
- Both Grafana replicas connected to the same database.
- Appropriate session handling.
- Replicas distributed across different Kubernetes nodes.
- Suitable storage and recovery configuration for the database.

With this architecture, both Grafana replicas can access the same dashboards, users, and datasource configurations.

This architecture can be implemented in a future phase.

### 5. Single Replica vs Multi-Replica Comparison

| Configuration | Single Replica | Multi-Replica |
|---|---|---|
| Grafana Pods | 1 | 2 or more |
| Database | Default SQLite | Shared PostgreSQL recommended |
| PVC | One PVC | Database storage managed separately |
| Configuration Complexity | Low | Higher |
| Dashboard Sharing | Single Grafana instance | Shared through common database |
| Pod Failure Recovery | Kubernetes recreates the pod | Another replica can serve requests |
| Node Failure Handling | Depends on storage and scheduling | Depends on replica distribution and database availability |
| HA Architecture | No | Possible with complete HA design |

**Important:** The current setup is a single-replica Grafana deployment with persistent storage. It should not be considered a fully highly available Grafana architecture.

## 16.3 Deployment Details

| Configuration | Value |
|---|---|
| Component | Grafana |
| Namespace | `monitoring` |
| Helm Release | `grafana` |
| Helm Repository | `grafana` |
| Helm Chart | `grafana/grafana` |
| Chart Version | `10.5.15` |
| Grafana App Version | `12.3.1` |
| Deployment Type | Deployment |
| Replicas | 1 |
| Service Type | ClusterIP |
| Service Port | 80 |
| Container Port | 3000 |
| Storage Type | PersistentVolumeClaim |
| PVC Name | `grafana` |
| Storage Capacity | 5Gi |
| Access Mode | ReadWriteOnce |
| StorageClass | `standard` |
| Datasource Configuration | Not configured |

## 16.4 Grafana Helm Repository Setup

The Grafana Helm repository is used to install and manage Grafana through Helm.

Add the Grafana repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update the Helm repositories:

```bash
helm repo update
```

Search for the Grafana chart:

```bash
helm search repo grafana/grafana
```

The chart version selected for this POC was:

```text
Chart Version: 10.5.15
App Version: 12.3.1
```

The repository was already configured, so Helm reported that it existed with the same configuration. The repository update completed successfully.

## 16.5 Fetch Grafana Default Values

Before creating the custom configuration, the default Helm chart values were fetched.

Command:

```bash
helm show values grafana/grafana \
  --version 10.5.15 > grafana-default-values.yaml
```

The command downloads the default values of the selected Grafana Helm chart into a local YAML file.

The file was approximately 61 KB.

Verify the file:

```bash
ls -lh grafana-default-values.yaml
```

### Important Default Configuration Sections

The following sections were inspected before creating the custom values file.

**1. Persistence**

```yaml
persistence:
  type: pvc
  enabled: false
  accessModes:
    - ReadWriteOnce
  size: 10Gi
```

By default, persistence was disabled.

We enabled persistence so that Grafana application data could be stored in a PersistentVolumeClaim instead of depending only on the container's writable filesystem.

**2. Service**

```yaml
service:
  enabled: true
  type: ClusterIP
  port: 80
  targetPort: 3000
```

The default service type was already ClusterIP.

We retained this configuration because Grafana only needs internal Kubernetes access for the current POC. External browser access can be provided through port-forwarding.

**3. Admin Configuration**

The chart provides an admin configuration and generates a Kubernetes Secret for the initial credentials when no existing Secret is supplied.

The admin password can be retrieved from the generated Secret.

**4. Datasources**

```yaml
datasources: {}
```

By default, no datasource was configured through the chart values.

We kept datasources empty in the current phase. Tempo, Loki, and VictoriaMetrics integration will be configured separately in a later step.

## 16.6 Create Grafana Directory

Grafana configuration files are maintained in a separate directory.

```bash
mkdir -p ~/ha-observability-poc/grafana

cd ~/ha-observability-poc/grafana
```

The directory contains:

```text
grafana/
├── grafana-default-values.yaml
├── grafana-values.yaml
└── grafana-rendered.yaml
```

- `grafana-default-values.yaml`: Default values downloaded from the Helm chart.
- `grafana-values.yaml`: Custom configuration used for Grafana deployment.
- `grafana-rendered.yaml`: Rendered Kubernetes resources generated through Helm template.

## 16.7 Grafana Custom Configuration

**Configuration File:**

`grafana/grafana-values.yaml`

Create the file:

```bash
nano grafana-values.yaml
```

Add the following configuration:

```yaml
replicas: 1

persistence:
  type: pvc
  enabled: true
  accessModes:
    - ReadWriteOnce
  size: 5Gi

service:
  enabled: true
  type: ClusterIP
  port: 80
  targetPort: 3000
```

### 16.7.1 Configuration Explanation

| Configuration | Explanation |
|---|---|
| `replicas: 1` | Creates one Grafana pod. |
| `persistence.type: pvc` | Uses PersistentVolumeClaim for storage. |
| `persistence.enabled: true` | Enables persistent storage. |
| `accessModes: ReadWriteOnce` | Allows read-write mounting from one node at a time. |
| `size: 5Gi` | Requests 5Gi storage for Grafana. |
| `service.enabled: true` | Creates a Kubernetes Service. |
| `type: ClusterIP` | Exposes Grafana internally within the cluster. |
| `port: 80` | Service listens on port 80. |
| `targetPort: 3000` | Forwards traffic to the Grafana application port. |

### Why Persistence Is Enabled

Grafana stores its application data in its data directory.

Without persistent storage, data stored only in the container's writable filesystem may be lost when the pod is recreated.

With persistence enabled, the Grafana data directory is mounted from the PVC.

Therefore, if the Grafana pod is deleted and recreated on a node where its local volume is accessible, the existing data can be retained.

**Storage limitation:** The Kind `standard` StorageClass uses local-path storage. The PVC does not automatically provide cross-node storage replication. If the node hosting the local volume becomes unavailable, the volume may not be accessible from another node.

## 16.8 Validate Grafana Helm Configuration

Before applying the configuration, Helm templates were rendered to inspect the Kubernetes resources.

Command:

```bash
helm template grafana grafana/grafana \
  --version 10.5.15 \
  --namespace monitoring \
  -f grafana-values.yaml > grafana-rendered.yaml
```

This generates Kubernetes YAML resources without installing them.

Verify the Deployment and PVC resources:

```bash
grep -nE 'kind: (Deployment|StatefulSet|PersistentVolumeClaim)|claimName:|replicas:' grafana-rendered.yaml
```

During the initial two-replica evaluation, the rendered output showed two replicas referencing one PVC.

After reviewing this configuration, the replica count was changed to one.

The final configuration was kept at:

```yaml
replicas: 1
```

## 16.9 Upgrade Existing Grafana Helm Release

Grafana was already installed as a Helm release named `grafana`.

Therefore, we used `helm upgrade` instead of `helm install`.

Check the existing release:

```bash
helm list -n monitoring
```

Upgrade Grafana using the custom values file:

```bash
helm upgrade grafana grafana/grafana \
  --version 10.5.15 \
  --namespace monitoring \
  -f grafana-values.yaml
```

**Observed Helm Result:**

```text
Release "grafana" has been upgraded. Happy Helming!

STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
```

Helm also displayed a warning that the selected Grafana chart is deprecated.

The warning did not prevent the upgrade from completing successfully.

## 16.10 Grafana PersistentVolumeClaim Verification

After the upgrade, the Grafana PVC was checked.

Command:

```bash
kubectl get pvc -n monitoring
```

**Observed Result:**

```text
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
grafana   Bound    pvc-a5f59077-dc07-4f65-ad0e-252305decfbf   5Gi        RWO            standard
```

### PVC Verification Details

| Configuration | Observed Result |
|---|---|
| PVC Name | `grafana` |
| Status | Bound |
| Capacity | 5Gi |
| Access Mode | RWO |
| StorageClass | standard |

The PVC was successfully provisioned and bound to a PersistentVolume.

The `standard` StorageClass in this Kind environment uses local-path storage.

## 16.11 Grafana Pod Verification

Check Grafana pods:

```bash
kubectl get pods -n monitoring -o wide
```

**Observed Result:**

```text
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE
grafana-74985977c5-fh97r   1/1     Running   0          94s   10.244.2.19   ha-observability-worker
```

### Deployment Verification

| Configuration | Expected Result | Observed Result |
|---|---|---|
| Grafana Replicas | 1 | 1 |
| Pod Status | Running | Running |
| Ready Containers | 1/1 | 1/1 |
| Restart Count | 0 | 0 |
| Node Placement | One eligible node | ha-observability-worker |
| Persistent Storage | Bound | Bound |
| Storage Capacity | 5Gi | 5Gi |

The Grafana pod was successfully scheduled on:

```text
ha-observability-worker
```

## 16.12 Verify Deployment Rollout

Check the Grafana Deployment:

```bash
kubectl get deployment -n monitoring
```

Verify the rollout:

```bash
kubectl rollout status deployment/grafana -n monitoring
```

**Observed Result:**

```text
deployment "grafana" successfully rolled out
```

This confirms that the updated Grafana Deployment completed successfully.

## 16.13 Grafana Service Verification

Check the Grafana Service:

```bash
kubectl get svc -n monitoring
```

The Helm release notes provide the internal service DNS name:

```text
grafana.monitoring.svc.cluster.local
```

The service exposes port 80 and forwards requests to Grafana's application port 3000.

| Configuration | Value |
|---|---|
| Service Name | grafana |
| Namespace | monitoring |
| Service Type | ClusterIP |
| Service Port | 80 |
| Target Port | 3000 |

## 16.14 Access Grafana Through Port Forwarding

Since Grafana is exposed through a ClusterIP Service, it can be accessed locally using Kubernetes port-forwarding.

Execute:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Keep the terminal running.

Grafana can then be accessed in the browser:

```text
http://localhost:3000
```

### Login Credentials

Username:

```text
admin
```

Retrieve the generated admin password:

```bash
kubectl get secret grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

Use the retrieved password to log in to Grafana.

**Note:** Browser login and datasource integration should be marked complete only after they have been tested.

## 16.15 Grafana Data Persistence and Recovery

The current deployment uses one Grafana replica with one 5Gi PVC.

The PVC is mounted at Grafana's data directory.

The expected recovery behavior is:

1. Grafana pod is running.
2. Grafana stores its application data in the mounted PVC.
3. The pod is deleted or recreated.
4. Kubernetes creates a replacement pod.
5. The replacement pod mounts the existing PVC.
6. Existing Grafana data can be accessed again.

This recovery behavior still needs to be verified through a controlled pod deletion and recreation test.

**Important Storage Limitation:**

The current local-path storage is tied to the node where the volume is provisioned. Pod recreation on the same node can use the existing data, but node failure recovery is not guaranteed.

Persistent storage protects against normal pod/container recreation. It does not by itself provide node-level storage HA.

## 16.16 Useful Verification and Troubleshooting Commands

| Task | Command |
|---|---|
| Check Grafana pods | `kubectl get pods -n monitoring -o wide` |
| Check Grafana Deployment | `kubectl get deployment -n monitoring` |
| Check Grafana Service | `kubectl get svc -n monitoring` |
| Check Grafana PVC | `kubectl get pvc -n monitoring` |
| Check Grafana PV | `kubectl get pv` |
| Check Grafana logs | `kubectl logs -n monitoring deployment/grafana` |
| Describe Grafana pod | `kubectl describe pod -n monitoring -l app.kubernetes.io/name=grafana` |
| Check Helm release | `helm list -n monitoring` |
| Check Helm values | `helm get values grafana -n monitoring -o yaml` |
| Check rollout status | `kubectl rollout status deployment/grafana -n monitoring` |
| Port-forward Grafana | `kubectl port-forward -n monitoring svc/grafana 3000:80` |
| Retrieve admin password | `kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" \| base64 --decode; echo` |

## 16.17 Current Grafana Architecture

The current Grafana deployment architecture is:

```text
                Kind Kubernetes Cluster
                    ha-observability
                            |
                            |
                    monitoring Namespace
                            |
                     Grafana Service
                        ClusterIP
                         Port 80
                            |
                            v
                     Grafana Deployment
                       Replicas: 1
                            |
                            v
                       Grafana Pod
                            |
                            |
                   Persistent Volume Claim
                        PVC: grafana
                         Capacity: 5Gi
                         Access: RWO
                            |
                            v
                     Local-Path Storage
                      StorageClass:
                        standard
```

Browser access:

```text
Local Browser
     |
     v
localhost:3000
     |
     v
kubectl port-forward
     |
     v
Grafana Service :80
     |
     v
Grafana Pod :3000
```

## 16.18 Final Deployment Result

| Component / Verification | Expected Result | Observed Result |
|---|---|---|
| Helm Release | Deployed | Deployed |
| Chart Version | 10.5.15 | 10.5.15 |
| Grafana App Version | 12.3.1 | 12.3.1 |
| Grafana Replicas | 1 | 1 |
| Grafana Pod | Running | 1/1 Running |
| Restart Count | 0 | 0 |
| PVC | Bound | Bound |
| Storage Capacity | 5Gi | 5Gi |
| StorageClass | standard | standard |
| Service Type | ClusterIP | Configured |
| Deployment Rollout | Successful | Successful |
| Datasource Configuration | Not configured in current phase | Pending |
| Pod Recovery Test | Pending | Not yet performed |
| Multi-Replica HA | Future phase | Not implemented |

