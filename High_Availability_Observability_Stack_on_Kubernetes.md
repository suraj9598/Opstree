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

