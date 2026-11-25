# vCluster Monitoring Validation and Production Rollout Guide
## coe-dev Reference Setup and coe-prod Enablement

**Document purpose:** This document records the monitoring configuration validated for `coe-dev`, the monitoring gaps identified in `coe-prod`, and the proposed steps to enable equivalent host-node metrics collection in `coe-prod`.

**Status:** Discovery and configuration-level validation completed. No production RBAC or vmagent configuration changes have been applied as part of this validation.

---

## 1. Executive Summary

The objective is to collect vCluster workload and host-node metrics for both `coe-dev` and `coe-prod` and make them available in VictoriaMetrics/Grafana.

The `coe-dev` environment is the working reference. Its vmagent configuration contains three scrape jobs:

1. `kube-state-metrics`
2. `vcluster-cadvisor`
3. `vcluster-kubelet-volume`

The `coe-prod` vmagent configuration currently contains only the `kube-state-metrics` job. The cAdvisor and kubelet volume-statistics jobs are missing. In addition, the existing shared `vcluster-node-proxy` ClusterRoleBinding currently references only the `coe-dev` control-plane ServiceAccount.

To bring `coe-prod` to parity with `coe-dev`, two configuration areas need to be addressed:

- **RBAC:** Authorize the `coe-prod` vCluster control-plane ServiceAccount to use the shared `vcluster-node-proxy` ClusterRole.
- **vmagent scrape configuration:** Add the cAdvisor and kubelet volume scrape jobs to the original `coe-prod` vmagent configuration source, while preserving the existing kube-state-metrics job and `cluster: "coe-prod"` external label.

The RBAC change and scrape configuration change serve different purposes. RBAC permits the vCluster API proxy path to access host node metrics endpoints; scrape jobs tell vmagent which endpoints and metrics to collect. Both are required for the proposed monitoring flow.

---

## 2. Scope

### Included

- Inspecting host-side ServiceAccounts for `coe-dev` and `coe-prod`.
- Inspecting vmagent pods and their associated ConfigMaps.
- Reviewing the shared `vcluster-node-proxy` ClusterRole and its ClusterRoleBinding.
- Comparing vmagent scrape jobs between the two environments.
- Checking the presence of representative metrics in Grafana.
- Defining the proposed RBAC and scrape configuration changes.
- Providing pre-change checks, rollout, validation, and rollback steps.
- Reviewing the TLS configuration used by the scrape jobs.

### Not included

- Changing the existing Grafana dashboards.
- Replacing VictoriaMetrics, vmagent, or kube-state-metrics.
- Investigating unrelated Kine/syncer log messages.
- Applying production changes. The current work is discovery and configuration planning; changes must be performed by an authorized operator through the approved deployment process.

---

## 3. Monitoring Architecture and Request Flow

The proposed design uses vmagent running inside each vCluster to scrape metrics through that vCluster's Kubernetes API Server.

### 3.1 Components

| Component | Purpose |
|---|---|
| vCluster | Provides the Kubernetes API endpoint used by the in-vCluster vmagent |
| vmagent | Discovers node targets, scrapes metrics, and remote-writes samples |
| vCluster control-plane ServiceAccount | Identity used by the vCluster API/proxy path for host node-proxy access |
| `vcluster-node-proxy` ClusterRole | Defines the host-side node and node-proxy permissions |
| kubelet | Exposes kubelet and cAdvisor metrics on host nodes |
| kube-state-metrics | Exposes Kubernetes object-state metrics |
| VictoriaMetrics | Receives and stores metrics |
| Grafana | Queries and visualizes the metrics |

### 3.2 cAdvisor and kubelet volume request flow

1. vmagent uses Kubernetes service discovery with `role: node` to discover node targets.
2. The scrape configuration rewrites the target address to `kubernetes.default.svc:443`, which is the Kubernetes API Service visible inside the vCluster.
3. The scrape configuration sets the metrics path to one of the following:
   - `/api/v1/nodes/<node-name>/proxy/metrics/cadvisor`
   - `/api/v1/nodes/<node-name>/proxy/metrics`
4. vmagent sends the request to the vCluster API Server using its projected ServiceAccount token and configured CA file.
5. The vCluster API/proxy path accesses the corresponding host kubelet endpoint. The host-side authorization must permit the required node-proxy access.
6. vmagent scrapes the returned metrics and remote-writes them to VictoriaMetrics.
7. Grafana queries VictoriaMetrics and displays the metrics.

**Important identity distinction:** The vmagent workload pod's host-side ServiceAccount and the vCluster control-plane ServiceAccount are different identities. In the inspected configuration, the node-proxy ClusterRoleBinding subject is the vCluster control-plane ServiceAccount (`vc-coe-dev-vcluster`), not the vmagent workload ServiceAccount (`vc-workload-coe-dev-vcluster`). Do not substitute one for the other without validating the actual vCluster proxy authorization model.

### 3.3 kube-state-metrics flow

The kube-state-metrics job uses a static service target:

`vm-kube-state-metrics.monitoring.svc.cluster.local.:8080`

This is separate from the host kubelet node-proxy jobs. Its successful scrape does not prove that cAdvisor or kubelet volume metrics are configured or authorized.

---

## 4. coe-dev: Validated Reference Configuration

### 4.1 ServiceAccounts and vmagent workload

The host namespace `coe-dev` contains:

| ServiceAccount | Role in the inspected setup |
|---|---|
| `vc-coe-dev-vcluster` | vCluster control-plane ServiceAccount |
| `vc-workload-coe-dev-vcluster` | Host-side ServiceAccount used by synced workload pods, including vmagent |

The inspected vmagent pod:
- Was in `Running` and `Ready` state at the time of validation.
- Used host ServiceAccount `vc-workload-coe-dev-vcluster`.
- Had the annotation `vcluster.loft.sh/service-account-name: vmagent`, identifying the original in-vCluster ServiceAccount.
- Used image `quay.io/opstree/victoriametrics-vmagent:v1.141.0`.
- Loaded configuration from ConfigMap `vmagent-vmagent-config-x-monitoring-x-coe-dev-vcluster`.

These observations describe the inspected state and should be rechecked before any future change.

### 4.2 Host-side RBAC

The shared ClusterRole `vcluster-node-proxy` was present with the following permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vcluster-node-proxy
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - nodes/stats
    verbs: [get, list, watch]
```

The existing ClusterRoleBinding `vcluster-node-proxy` references this ClusterRole and binds it to the `coe-dev` control-plane ServiceAccount:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vcluster-node-proxy
subjects:
  - kind: ServiceAccount
    name: vc-coe-dev-vcluster
    namespace: coe-dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vcluster-node-proxy
```

This confirms that the inspected binding authorizes the `coe-dev` control-plane ServiceAccount. It does not automatically authorize the `coe-prod` ServiceAccount.

### 4.3 vmagent scrape jobs

The `coe-dev` vmagent configuration has the following jobs:

| Job | Purpose |
|---|---|
| `kube-state-metrics` | Kubernetes object-state metrics |
| `vcluster-cadvisor` | Container and pod resource metrics exposed by cAdvisor |
| `vcluster-kubelet-volume` | Kubelet volume statistics, including PVC usage/capacity metrics |

The cAdvisor job uses:
- `scheme: https`
- The projected ServiceAccount CA file
- The projected ServiceAccount bearer token
- Kubernetes node discovery (`role: node`)
- API Server node-proxy path `/api/v1/nodes/<node>/proxy/metrics/cadvisor`
- A metric relabel rule dropping samples with an empty `pod` label

The kubelet volume job uses the same HTTPS, CA, bearer-token, and node-discovery pattern, with path `/api/v1/nodes/<node>/proxy/metrics`. Its metric relabel rule keeps metrics matching `kubelet_volume_stats_.*`.

The global configuration includes:
```yaml
global:
  scrape_interval: 30s
  external_labels:
    cluster: "coe-dev"
```

### 4.4 cAdvisor scrape job reference

```yaml
- job_name: vcluster-cadvisor
  scrape_interval: 30s
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  kubernetes_sd_configs:
    - role: node
  relabel_configs:
    - target_label: __address__
      replacement: kubernetes.default.svc:443
    - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
    - source_labels: [__meta_kubernetes_node_name]
      target_label: node
    - source_labels: [__meta_kubernetes_node_name]
      target_label: instance
    - target_label: job
      replacement: kubelet
    - target_label: metrics_path
      replacement: /metrics/cadvisor
  metric_relabel_configs:
    - source_labels: [pod]
      regex: ""
      action: drop
```

### 4.5 kubelet volume scrape job reference

```yaml
- job_name: vcluster-kubelet-volume
  scrape_interval: 30s
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  kubernetes_sd_configs:
    - role: node
  relabel_configs:
    - target_label: __address__
      replacement: kubernetes.default.svc:443
    - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/$1/proxy/metrics
    - source_labels: [__meta_kubernetes_node_name]
      target_label: node
    - source_labels: [__meta_kubernetes_node_name]
      target_label: instance
    - target_label: job
      replacement: kubelet
    - target_label: metrics_path
      replacement: /metrics
  metric_relabel_configs:
    - source_labels: [__name__]
      regex: kubelet_volume_stats_.*
      action: keep
```

**Configuration note:** These are reference job definitions based on the inspected `coe-dev` configuration. Before using them in `coe-prod`, compare the full current configuration and preserve any environment-specific settings.

---

## 5. coe-prod: Current Findings

### 5.1 ServiceAccounts and vmagent workload

The host namespace `coe-prod` contains:

| ServiceAccount | Role in the inspected setup |
|---|---|
| `vc-coe-prod-vcluster` | vCluster control-plane ServiceAccount |
| `vc-workload-coe-prod-vcluster` | Host-side ServiceAccount used by synced workload pods, including vmagent |

The inspected vmagent pod was:

`vmagent-vmagent-5d899c5455-lqjdr-x-monitoring-x-coe--2987a254ec`

At the time of inspection:
- The pod was `Running` and `Ready`.
- It had zero restarts in the inspected output.
- Its host ServiceAccount was `vc-workload-coe-prod-vcluster`.
- Its annotation was `vcluster.loft.sh/service-account-name: vmagent`.
- It used image `quay.io/opstree/victoriametrics-vmagent:v1.141.0`.
- Its ConfigMap was `vmagent-vmagent-config-x-monitoring-x-coe-prod-vcluster`.

A healthy vmagent pod only confirms that the process is running. It does not confirm that every expected scrape job exists or is succeeding.

### 5.2 Current coe-prod scrape configuration

The inspected `coe-prod` ConfigMap contained only:

```yaml
global:
  scrape_interval: 30s
  external_labels:
    cluster: "coe-prod"

scrape_configs:
  - job_name: kube-state-metrics
    scrape_interval: 30s
    static_configs:
      - targets:
          - vm-kube-state-metrics.monitoring.svc.cluster.local.:8080
```

The following jobs were missing:
- `vcluster-cadvisor`
- `vcluster-kubelet-volume`

This is a direct configuration gap: vmagent cannot scrape cAdvisor or kubelet volume metrics through jobs that are not configured.

### 5.3 Current host-side RBAC findings

The shared ClusterRole `vcluster-node-proxy` exists and includes:
- `nodes`
- `nodes/proxy`
- `nodes/metrics`
- `nodes/stats`

However, the existing ClusterRoleBinding named `vcluster-node-proxy` references only:

- ServiceAccount: `vc-coe-dev-vcluster`
- Namespace: `coe-dev`

The inspected `coe-prod` binding named `vc-coe-prod-vcluster-v-coe-prod` references a separate ClusterRole named `vc-coe-prod-vcluster-v-coe-prod`. That role includes other permissions such as PV, StorageClass, snapshot, services, endpoints, and `metrics.k8s.io/nodes`, but it does not include the same core `nodes`, `nodes/proxy`, `nodes/metrics`, and `nodes/stats` permissions from the shared node-proxy role.

**Important:** `metrics.k8s.io/nodes` is not equivalent to core `nodes/proxy`. Permission to read the Metrics API does not, by itself, establish permission to proxy requests to kubelet metrics endpoints.

### 5.4 Grafana metric checks

The following checks were performed for `coe-prod`:

```promql
container_cpu_usage_seconds_total{cluster="coe-prod"}
```

Result: no data was returned in the inspected check.

```promql
kubelet_volume_stats_used_bytes{cluster="coe-prod"}
```

Result: no data was returned in the inspected check.

```promql
up{cluster="coe-prod"}
```

Result: only the `kube-state-metrics` job was present with value `1`.

These results are consistent with the inspected vmagent configuration: kube-state-metrics is configured, while the cAdvisor and kubelet volume jobs are absent.

**Interpretation:** The missing scrape jobs are a confirmed configuration gap. The missing `coe-prod` node-proxy binding is an additional RBAC gap to address before enabling the node-proxy scrape jobs. The available evidence does not establish RBAC as the only cause of missing metrics.

---

## 6. Environment Comparison

| Area | coe-dev | coe-prod |
|---|---|---|
| vmagent pod | Running/Ready when inspected | Running/Ready when inspected |
| kube-state-metrics job | Present | Present |
| cAdvisor job | Present | Missing |
| kubelet volume job | Present | Missing |
| Global cluster label | `coe-dev` | `coe-prod` |
| Shared node-proxy ClusterRole | Present | Present |
| Shared binding subject | `vc-coe-dev-vcluster` in `coe-dev` | Not included in the shared binding |
| Grafana `up` check | Reference jobs validated in the working setup | Only kube-state-metrics observed |
| cAdvisor metrics | Confirmed available for coe-dev | No data in inspected check |
| kubelet volume metrics | Confirmed available for coe-dev | No data in inspected check |

---

## 7. Proposed coe-prod Changes

Two changes are proposed. They should be reviewed and applied through the approved production change process.

### 7.1 RBAC: authorize the coe-prod control-plane ServiceAccount

The existing shared ClusterRole can be reused. There are two valid ways to bind the `coe-prod` ServiceAccount.

#### Option A — Recommended: create a separate ClusterRoleBinding

This keeps the existing `coe-dev` binding unchanged and makes `coe-prod` access independently manageable.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vcluster-node-proxy-coe-prod
subjects:
  - kind: ServiceAccount
    name: vc-coe-prod-vcluster
    namespace: coe-prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vcluster-node-proxy
```

Apply only after approval from the cluster/platform owner:

```bash
kubectl apply -f vcluster-node-proxy-coe-prod.yaml
```

Verify:

```bash
kubectl get clusterrolebinding vcluster-node-proxy-coe-prod -o yaml
```

#### Option B — Add coe-prod to the existing binding

It is also technically valid to add a second subject to the existing `vcluster-node-proxy` ClusterRoleBinding:

```yaml
subjects:
  - kind: ServiceAccount
    name: vc-coe-dev-vcluster
    namespace: coe-dev
  - kind: ServiceAccount
    name: vc-coe-prod-vcluster
    namespace: coe-prod
```

This means one binding grants the shared role to both ServiceAccounts. If this option is selected, update the binding declaratively through the approved source of truth and verify that both subjects remain present.

**Recommendation:** Prefer Option A because it provides independent lifecycle management and rollback for `coe-prod`. Do not replace or remove the existing `coe-dev` subject.

### 7.2 vmagent: add the missing scrape jobs

Add the two reference jobs from Section 4 to the original `coe-prod` vmagent configuration source.

Preserve the existing global configuration:

```yaml
global:
  scrape_interval: 30s
  external_labels:
    cluster: "coe-prod"
```

Preserve the existing kube-state-metrics job and add:
- `vcluster-cadvisor`
- `vcluster-kubelet-volume`

The resulting structure should contain all three jobs:

```yaml
global:
  scrape_interval: 30s
  external_labels:
    cluster: "coe-prod"

scrape_configs:
  - job_name: kube-state-metrics
    scrape_interval: 30s
    static_configs:
      - targets:
          - vm-kube-state-metrics.monitoring.svc.cluster.local.:8080

  # Add the vcluster-cadvisor job from Section 4.4 here.

  # Add the vcluster-kubelet-volume job from Section 4.5 here.
```

The comments above are placeholders, not valid scrape jobs. The complete job definitions from Sections 4.4 and 4.5 must be inserted before deployment.

### 7.3 Update the source of truth, not only the mirrored ConfigMap

The inspected host-side ConfigMap is named:

`vmagent-vmagent-config-x-monitoring-x-coe-prod-vcluster`

It is a mirrored configuration associated with the vCluster workload. Directly editing only this mirrored ConfigMap may be temporary; the vCluster or Helm-managed source may recreate it and overwrite the manual change.

Before changing anything:
1. Identify the original vmagent configuration source inside `coe-prod` (for example, the Helm values/configuration managed by the deployment).
2. Identify the Helm release name, chart, chart version, and values file used by that environment.
3. Back up the current source configuration and rendered/current ConfigMap.
4. Make the change in the authoritative source.
5. Deploy through the approved release process.
6. Confirm the mirrored ConfigMap contains the intended configuration.

The exact Helm release name, chart version, and values file were not established during this validation. Therefore, a specific `helm upgrade` command is intentionally not included here.

---

## 8. Pre-change Checklist and Dry Run

Complete these checks before production rollout.

### 8.1 Confirm current objects

```bash
kubectl get serviceaccount -n coe-prod
kubectl get clusterrole vcluster-node-proxy -o yaml
kubectl get clusterrolebinding vcluster-node-proxy -o yaml
kubectl get configmap vmagent-vmagent-config-x-monitoring-x-coe-prod-vcluster -n coe-prod -o yaml
kubectl get pods -n coe-prod -o wide
```

Use the correct namespace/context for the actual deployment. The examples above reflect the inspected host-side objects.

### 8.2 Back up the current configuration

Save the current vmagent source values/configuration and the current ConfigMap YAML in the change record or an approved secure location. For example:

```bash
kubectl get configmap vmagent-vmagent-config-x-monitoring-x-coe-prod-vcluster \
  -n coe-prod -o yaml > coe-prod-vmagent-config-before.yaml
```

Also back up the authoritative Helm values or deployment configuration. A ConfigMap backup alone is not sufficient if it is generated from another source.

### 8.3 Validate the proposed YAML

For the RBAC manifest:

```bash
kubectl apply --dry-run=server -f vcluster-node-proxy-coe-prod.yaml
```

For the vmagent configuration, validate YAML syntax and vmagent/Prometheus scrape configuration compatibility using the validation tooling available in the deployment pipeline or an approved environment. A Kubernetes server-side dry run of a ConfigMap does not necessarily validate the embedded scrape configuration.

### 8.4 Check permissions

The operator performing the change needs appropriate permission to:
- Create or update the ClusterRoleBinding at cluster scope.
- Update the authoritative vmagent Helm/vCluster configuration.
- Perform the approved Helm release or deployment operation.
- Read pods, ConfigMaps, events, and logs for post-change validation.

The current inspection account was read-only, so it should not be assumed to have these write permissions.

---

## 9. Production Rollout Procedure

Perform during an approved change window and follow the platform team's change-management process.

### Step 1 — Confirm the change inputs

Confirm:
- Target vCluster: `coe-prod`.
- Control-plane ServiceAccount: `vc-coe-prod-vcluster`.
- Shared ClusterRole: `vcluster-node-proxy`.
- Authoritative vmagent configuration source.
- Exact Helm release/chart/version and values file, if Helm-managed.
- Rollback backup and responsible operator.

### Step 2 — Apply the RBAC change

If Option A is approved, create the separate binding:

```bash
kubectl apply -f vcluster-node-proxy-coe-prod.yaml
```

Verify the subject and role reference:

```bash
kubectl get clusterrolebinding vcluster-node-proxy-coe-prod -o yaml
```

Do not modify the shared ClusterRole unless a separate review identifies a missing permission. The current proposed change reuses the existing role.

### Step 3 — Update vmagent configuration

Add the two scrape jobs to the authoritative `coe-prod` configuration source. Preserve:
- Existing kube-state-metrics job.
- `cluster: "coe-prod"` external label.
- Existing remote-write settings and other deployment-specific settings.
- Existing scrape interval unless a reviewed requirement calls for a change.

Deploy through the established Helm/vCluster configuration workflow.

### Step 4 — Check rollout health

Check the vmagent pod and rollout status using the actual workload deployment name and namespace:

```bash
kubectl get pods -n coe-prod -o wide
```

If a Deployment name is confirmed:

```bash
kubectl rollout status deployment/<vmagent-deployment-name> -n coe-prod
```

Do not copy the placeholder literally; replace it with the verified Deployment name. If the workload is managed by a different controller, use that controller's rollout/status procedure.

### Step 5 — Confirm the active configuration

Inspect the generated/mirrored ConfigMap after deployment:

```bash
kubectl get configmap vmagent-vmagent-config-x-monitoring-x-coe-prod-vcluster \
  -n coe-prod -o yaml
```

Confirm that it includes:
- `kube-state-metrics`
- `vcluster-cadvisor`
- `vcluster-kubelet-volume`

Also confirm `cluster: "coe-prod"` remains unchanged.

### Step 6 — Check vmagent logs and targets

Inspect vmagent logs for scrape, authentication, authorization, TLS, and remote-write errors:

```bash
kubectl logs <vmagent-pod-name> -n coe-prod --since=15m
```

Replace the placeholder with the current vmagent pod name. Where the vmagent UI or target endpoint is available through the approved access path, verify that both new jobs are discovered and targets are healthy.

### Step 7 — Validate in Grafana

Run the following queries in the Grafana datasource backed by VictoriaMetrics.

**Scrape health by job:**

```promql
up{cluster="coe-prod"}
```

Expected: the kube-state-metrics job and the new node-proxy scrape jobs should appear. For discovered targets, a value of `1` indicates a successful scrape; `0` indicates a scrape failure.

**Container CPU metric:**

```promql
container_cpu_usage_seconds_total{cluster="coe-prod"}
```

Expected: data should be returned for applicable containers/pods.

**Container memory metric:**

```promql
container_memory_working_set_bytes{cluster="coe-prod"}
```

Expected: data should be returned for applicable containers/pods.

**Kubelet volume usage:**

```promql
kubelet_volume_stats_used_bytes{cluster="coe-prod"}
```

Expected: data should be returned for volumes for which kubelet exposes the relevant statistics.

**Kubelet volume capacity:**

```promql
kubelet_volume_stats_capacity_bytes{cluster="coe-prod"}
```

Expected: data should be returned where the relevant volume statistics are available.

Metric availability can depend on workload type, volume type, node state, and kubelet behavior. Validate against known active workloads/PVCs rather than treating an empty result for an unused resource as automatic proof of failure.

### Step 8 — Record the outcome

Record:
- Change ticket/reference and time.
- RBAC option selected and resulting binding.
- Source configuration/release updated.
- vmagent rollout status.
- Target health for each job.
- Grafana query results.
- Any warnings, exceptions, or follow-up work.

---

## 10. Troubleshooting Guide

| Symptom | What to check |
|---|---|
| `up` has only kube-state-metrics | Confirm both new jobs are present in the active vmagent configuration |
| New job appears with `up = 0` | Inspect vmagent logs and target last-error details |
| `403 Forbidden` | Review the identity used by the vCluster API/proxy path and the host-side RBAC binding/role |
| `401 Unauthorized` | Check projected ServiceAccount token availability and the identity used for the request |
| TLS or x509 error | Check CA file, API Service certificate, SAN, and TLS settings |
| No node targets discovered | Check Kubernetes node service discovery and the relevant discovery permissions |
| Targets healthy but no expected metric | Check metric relabel rules, metric name, labels, and whether the relevant workload/volume exists |
| Metrics reach vmagent but not VictoriaMetrics | Check remote-write URL, connectivity, and remote-write errors |
| Grafana query returns no data | Confirm datasource, time range, `cluster` label, metric name, and whether samples reached VictoriaMetrics |

Do not infer the cause from a single symptom. Use vmagent target status and logs together with RBAC and active configuration checks.

---

## 11. TLS Security Review

The inspected cAdvisor and kubelet-volume jobs use:

```yaml
tls_config:
  ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  insecure_skip_verify: true
```

`insecure_skip_verify: true` disables certificate verification. Although HTTPS is used, the client does not validate the server certificate in the normal way. This weakens protection against connecting to an unintended or intercepted endpoint.

### Recommended review

1. Confirm that the projected CA file exists inside the vmagent container.
2. Inspect the CA certificate/bundle using an approved diagnostic method.
3. Confirm which certificate authority signs the vCluster API Service certificate.
4. Confirm the certificate SAN matches the hostname vmagent uses (`kubernetes.default.svc`), or document the approved `server_name` setting if a different name is used.
5. Confirm the configured CA is the correct trust chain for that endpoint; do not assume the host API Server CA and vCluster service CA are interchangeable.
6. In a controlled environment, test certificate verification with `insecure_skip_verify: false`.
7. Check vmagent target health and logs after the test.
8. Only after successful validation, plan a production change to enable verification through the normal change process.

Do not switch `insecure_skip_verify` to `false` directly in production without first confirming the certificate chain and hostname. Otherwise, scrapes may fail with certificate verification errors.

---

## 12. Rollback Plan

Rollback should be limited to the changes made for this rollout.

### 12.1 RBAC rollback

If the separate binding Option A was used, delete only the new `coe-prod` binding:

```bash
kubectl delete clusterrolebinding vcluster-node-proxy-coe-prod
```

Do not delete:
- The shared `vcluster-node-proxy` ClusterRole.
- The existing `vcluster-node-proxy` ClusterRoleBinding used by `coe-dev`.

If Option B was used, remove only the `coe-prod` subject from the existing binding through the approved declarative source. Keep the `coe-dev` subject intact.

### 12.2 vmagent configuration rollback

1. Restore the backed-up authoritative vmagent values/configuration.
2. Deploy the previous known-good configuration through the same Helm/vCluster deployment process.
3. Confirm the generated ConfigMap returns to the previous state.
4. Confirm vmagent returns to a healthy state.
5. Confirm the existing kube-state-metrics job still works.

Do not rely only on manually restoring the mirrored ConfigMap if it is generated by Helm or vCluster; the next reconciliation may overwrite the manual change.

### 12.3 Rollback verification

Run:

```promql
up{cluster="coe-prod"}
```

Confirm the environment has returned to the expected pre-change scrape state. Record the rollback reason and results in the change record.

---

## 13. Security and Operational Considerations

- The node-proxy role is cluster-scoped because it grants access to node proxy endpoints. Apply it only to the intended vCluster control-plane ServiceAccount.
- Reusing the existing role avoids duplicating permission definitions, but the permissions granted by that role should still be reviewed by the platform/security owner.
- A separate ClusterRoleBinding provides a clearer independent rollback path for `coe-prod`.
- Keep the `coe-dev` binding unchanged during the `coe-prod` rollout.
- Make changes in the authoritative Helm/vCluster source, not only in generated ConfigMaps.
- Keep the `cluster` external label unique and correct so metrics from environments remain distinguishable.
- Review TLS certificate verification before production hardening.
- Monitor scrape health and remote-write health after rollout.
- Document the change and rollback evidence.

---

## 14. Current Status and Open Items

### Completed

- Inspected `coe-dev` and `coe-prod` host-side ServiceAccounts.
- Inspected vmagent pod identity, status, image, and associated ConfigMap.
- Inspected the shared `vcluster-node-proxy` ClusterRole.
- Inspected the existing `vcluster-node-proxy` ClusterRoleBinding.
- Compared `coe-dev` and `coe-prod` scrape configurations.
- Confirmed cAdvisor and kubelet volume jobs are present in the inspected `coe-dev` configuration and absent from the inspected `coe-prod` configuration.
- Checked Grafana queries for `coe-prod`; only kube-state-metrics was observed in the `up` query.
- Confirmed cAdvisor and kubelet volume metrics were available for `coe-dev` in the validation performed.

### Pending before production enablement

- Platform/admin approval for the `coe-prod` node-proxy RBAC binding.
- Confirmation of the authoritative vmagent configuration source.
- Confirmation of the exact Helm release name, chart version, and values file (if Helm-managed).
- Backup and validation of the proposed configuration.
- Application of the approved RBAC and vmagent changes.
- Post-change target, log, and Grafana validation.
- TLS certificate verification review and a controlled test of `insecure_skip_verify: false`.

