# HyperDX / ClickStack on Kubernetes --- Setup Guide

## 1. Overview

This document explains HyperDX from the basics, the architecture
deployed in this environment, the setup steps performed, verification
commands, and troubleshooting notes.

**Design decision:** this installation reuses the existing ClickHouse
cluster in the `clickhouse` namespace. The ClickStack Helm values set
`clickhouse.enabled: false`. Do not deploy another ClickHouse instance
or delete/change the existing ClickHouse PVCs as part of this setup.

## 2. What is HyperDX?

HyperDX is an observability application that provides a web interface
for searching and investigating telemetry such as logs, traces, and
application events. Engineers can search log messages, filter by
Kubernetes metadata, inspect event details, build dashboards, and
investigate operational issues.

Example: if an application is failing, an engineer can open the Logs
view, filter by namespace/pod, search for an error, inspect timestamps
and event fields, and use the findings to troubleshoot.

### Main components

-   **HyperDX application:** the web UI and application API used by
    engineers.
-   **ClickHouse:** analytical database that stores and queries
    telemetry data.
-   **OpenTelemetry Collector:** receives telemetry and exports it to
    configured backends.
-   **MongoDB:** stores HyperDX application-related state/metadata. It
    is not the telemetry database in this architecture.
-   **MongoDB Kubernetes Operator:** manages the MongoDB custom resource
    and related Kubernetes resources.


## Why does HyperDX need MongoDB?

MongoDB is a supporting database for the HyperDX application itself. In this deployment, ClickHouse and MongoDB have different responsibilities:

- **ClickHouse** stores and queries the observability data, such as logs and other telemetry.
- **MongoDB** stores HyperDX application-related state/metadata that the application needs to operate.
- **HyperDX** is the UI/application layer that lets an engineer search and investigate the telemetry stored in ClickHouse.

MongoDB is therefore not a replacement for ClickHouse, and it is not where this setup's main log-query workload is intended to live. Even when ClickHouse is already available, HyperDX can still require MongoDB for its own application state.

### What happened in our setup?

The ClickStack release included a MongoDB custom resource, but the MongoDB Kubernetes Operator was not initially reconciling it into a healthy MongoDB pod. We installed the operator in the separate `mongodb-operator` namespace and fixed the specific Kubernetes RBAC errors shown in its logs and the MongoDB agent logs.

After those fixes, the MongoDB pod `hyperdx-clickstack-mongodb-0` reached `2/2 Running`. This was important because the HyperDX application depends on MongoDB being available for its application-level state.

## 3. Architecture

``` text
Engineer browser
    |
    | http://localhost:3000
    | kubectl port-forward
    v
+-----------------------------------+
| HyperDX Web Application           |
| Namespace: hyperdx                |
| Service: hyperdx-clickstack-app   |
| Ports: 3000 (UI), 4320 (API)      |
+----------------+------------------+
                 |
                 | application state / metadata
                 v
+-----------------------------------+
| MongoDB                           |
| Pod: hyperdx-clickstack-mongodb-0 |
| Namespace: hyperdx                |
| Port: 27017                       |
+-----------------------------------+

Telemetry sources (when configured)
                 |
                 v
+-----------------------------------+
| OpenTelemetry Collector           |
| Namespace: hyperdx                |
| OTLP: 4317/TCP and 4318/TCP       |
+----------------+------------------+
                 |
                 | ClickHouse exporter (native TCP)
                 | port 9000
                 v
+-----------------------------------+
| Existing ClickHouse cluster       |
| Namespace: clickhouse             |
| Headless service:                 |
| clickhouse-clickhouse-headless    |
+----------------+------------------+
                 ^
                 |
                 | HyperDX data-source queries over HTTP
                 | port 8123
+----------------+------------------+
| clickhouse-http service           |
| clickhouse namespace              |
+-----------------------------------+
```

### Architecture explanation

1.  The engineer opens HyperDX through a local port-forward. This is
    local access, not a public ingress.
2.  The HyperDX application uses MongoDB for its application
    state/metadata.
3.  Telemetry sources can send data to the OpenTelemetry Collector using
    OTLP, if those sources are configured.
4.  The Collector exports telemetry to the existing ClickHouse cluster
    using the native TCP protocol on port `9000`.
5.  HyperDX queries ClickHouse through its HTTP service on port `8123`
    to show searchable events.
6.  ClickHouse and HyperDX run in separate namespaces. Reusing
    ClickHouse avoids creating a second database deployment for this
    POC.

The diagram describes configured components and endpoints; it does not
mean every possible telemetry source is already instrumented or
exporting data.

## 4. Environment and endpoints

  -------------------------------------------------------------------------------------------------------------------------------
  Item                                Value
  ----------------------------------- -------------------------------------------------------------------------------------------
  Helm release                        `hyperdx`

  Chart                               `clickstack/clickstack`

  Chart version                       `3.4.0`

  Application version                 `2.39.1`

  HyperDX namespace                   `hyperdx`

  Existing ClickHouse namespace       `clickhouse`

  MongoDB operator namespace          `mongodb-operator`

  Values file                         `~/helm/hyperdx-values.yaml`

  HyperDX URL                         `http://localhost:3000`

  HyperDX service                     `hyperdx-clickstack-app`

  MongoDB service                     `hyperdx-clickstack-mongodb-svc`

  Collector service                   `hyperdx-otel-collector`

  ClickHouse HTTP endpoint            `http://clickhouse-http.clickhouse.svc.cluster.local:8123`

  ClickHouse native endpoint          `tcp://clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9000?dial_timeout=10s`
  -------------------------------------------------------------------------------------------------------------------------------

The values file contains `clickhouse.enabled: false`, so ClickStack does
not install a second ClickHouse. The HyperDX data source uses the HTTP
endpoint; the Collector exporter uses the native TCP endpoint. These
Kubernetes DNS names are for in-cluster communication.

**Credentials:** do not put passwords or tokens in this README or commit
them to Git. The existing `default` ClickHouse user was tested without a
password in this environment; verify the security settings before using
this assumption elsewhere.

## 5. Prerequisites

-   Correct Kubernetes context and working `kubectl`.
-   Helm installed.
-   Existing ClickHouse cluster healthy in namespace `clickhouse`.
-   Permission to create namespaces, Helm releases, service accounts,
    Roles and RoleBindings.
-   Helm repositories/charts available locally.

Initial checks:

``` bash
kubectl config current-context
kubectl get pods -n clickhouse
kubectl get svc -n clickhouse
```

Always confirm the active Kubernetes context before making changes.

## 6. Setup steps performed

### Step 1 --- Prepare the values file

Configuration file:

``` bash
less ~/helm/hyperdx-values.yaml
```

The key setting is:

``` yaml
clickhouse:
  enabled: false
```

This reuses the existing ClickHouse. The HyperDX data source and
Collector exporter were pointed at the existing ClickHouse service
endpoints listed above.

### Step 2 --- Install MongoDB Kubernetes Operator

The operator was installed in its own namespace:

``` bash
helm install mongodb-operator mongodb/mongodb-kubernetes \
  --version 1.7.0 \
  --namespace mongodb-operator \
  --create-namespace
```

Check it:

``` bash
kubectl get pods -n mongodb-operator
kubectl get deployment -n mongodb-operator
```

The deployment was named `mongodb-kubernetes-operator`, and its watch
scope was configured for `hyperdx`.

An operator watches custom resources and reconciles them into Kubernetes
resources. Installing an operator alone does not guarantee the database
instance is healthy; its custom resource, permissions, pods, and storage
must also be checked.

### Step 3 --- Fix operator RBAC for MongoDB custom resources

The operator logs showed an RBAC error when listing
`mongodbsearch.mongodb.com` in `hyperdx`. A namespace-scoped
Role/RoleBinding was created for the operator, and the Role was updated
to include the required `mongodb.com` resources.

Permission check:

``` bash
kubectl auth can-i list mongodbsearch.mongodb.com \
  -n hyperdx \
  --as=system:serviceaccount:mongodb-operator:mongodb-kubernetes-operator
```

Expected after the fix:

``` text
yes
```

Use least-privilege RBAC. Do not grant broad cluster-admin permissions
just to silence an error.

### Step 4 --- Create the MongoDB application service account

MongoDB pod creation was blocked because this service account did not
exist:

``` text
hyperdx/mongodb-kubernetes-appdb
```

It was created:

``` bash
kubectl create serviceaccount mongodb-kubernetes-appdb -n hyperdx
kubectl get serviceaccount mongodb-kubernetes-appdb -n hyperdx
```

### Step 5 --- Fix MongoDB agent permissions

The MongoDB pod's `mongod` container started, but `mongodb-agent` did
not initially become ready. Logs exposed permission errors in sequence:
reading Secret `hyperdx-clickstack-mongodb-config`, getting pods, then
patching pods.

A namespace-scoped Role/RoleBinding named
`hyperdx-mongodb-config-reader` was created so service account
`mongodb-kubernetes-appdb` could read the named configuration Secret.

A namespace-scoped Role/RoleBinding named `hyperdx-mongodb-pod-reader`
was created for pod access. The Role was updated to permit `get` and
`patch` on pods:

``` bash
kubectl patch role hyperdx-mongodb-pod-reader -n hyperdx \
  --type=merge \
  -p '{"rules":[{"apiGroups":[""],"resources":["pods"],"verbs":["get","patch"]}]}'
```

Verify:

``` bash
kubectl auth can-i get pods -n hyperdx \
  --as=system:serviceaccount:hyperdx:mongodb-kubernetes-appdb
kubectl auth can-i patch pods -n hyperdx \
  --as=system:serviceaccount:hyperdx:mongodb-kubernetes-appdb
```

Both returned `yes`. The agent then reported it had reached its goal
state, and the MongoDB pod became `2/2 Running`.

These permissions addressed the errors observed in this environment.
Before production use, review the operator version's documented RBAC
requirements and keep permissions as narrow as possible.

### Step 6 --- Install/upgrade ClickStack with Helm

The release was installed/upgraded with:

``` bash
helm upgrade --install hyperdx clickstack/clickstack \
  --namespace hyperdx \
  -f ~/helm/hyperdx-values.yaml
```

For reproducible future deployments, pin the chart version:

``` bash
helm upgrade --install hyperdx clickstack/clickstack \
  --version 3.4.0 \
  --namespace hyperdx \
  -f ~/helm/hyperdx-values.yaml
```

Use the pinned command only after confirming the chart repository and
version are available in the local Helm configuration.

Check release status:

``` bash
helm status hyperdx -n hyperdx
helm get values hyperdx -n hyperdx
```

### Step 7 --- Confirm ClickHouse integration

The Collector logs showed a successful ClickHouse connection and
completed schema/seed operations, including:

``` text
goose: up to current file version: 7
Schema seed completed successfully
```

This confirms the Collector reached ClickHouse and completed those setup
operations. It does not prove that every intended telemetry source is
sending data.

### Step 8 --- Check pods and services

At the last validation, these pods were healthy:

``` text
hyperdx-clickstack-app-...        1/1 Running
hyperdx-clickstack-mongodb-0      2/2 Running
hyperdx-otel-collector-...        1/1 Running
```

The Collector had historical restarts during troubleshooting, but was
running and ready at the last check. Pod suffixes can change after
rollouts.

Commands:

``` bash
kubectl get pods -n hyperdx -o wide
kubectl get svc -n hyperdx
kubectl get pvc -n hyperdx
```

### Step 9 --- Open the HyperDX UI

Run this in a terminal and keep it open:

``` bash
kubectl port-forward -n hyperdx svc/hyperdx-clickstack-app 3000:3000
```

Open:

``` text
http://localhost:3000
```

The login page appeared and the user entered an email and password. The
HyperDX Logs search page then opened successfully.

If local port 3000 is busy, use a different local port:

``` bash
kubectl port-forward -n hyperdx svc/hyperdx-clickstack-app 3001:3000
```

Then open `http://localhost:3001`.

### Step 10 --- Verify logs in the UI

The Logs page showed approximately `6,987 Results` at the time of the
screenshot, with event bodies containing ClickHouse/ClickHouse Keeper
messages.

Verification steps:

1.  Open **Search** and select **Logs**.
2.  Confirm that results appear in the chosen time range.
3.  Expand the `k8s.namespace.name` filter.
4.  Select `clickhouse` if available, then click **Run**.
5.  Open an event and inspect its timestamp and fields.
6.  If no results appear, remove filters and widen the time range before
    concluding that ingestion is broken.

The count changes with incoming data and time range, so `6,987` is an
observation, not a fixed expected count.


## RBAC permissions added during troubleshooting

Kubernetes RBAC (Role-Based Access Control) determines which Kubernetes resources a service account can access and which actions it can perform. We used the error messages in the operator and MongoDB agent logs to identify missing permissions. The permissions below describe the fixes performed during this setup.

### 1. MongoDB Operator permissions

- **Service account:** `system:serviceaccount:mongodb-operator:mongodb-kubernetes-operator`
- **Namespace where the Role/RoleBinding was created:** `hyperdx`
- **Observed issue:** the operator could not list `mongodbsearch.mongodb.com` in the `hyperdx` namespace.
- **Fix performed:** created/updated a namespace-scoped Role and RoleBinding for the operator, including the required MongoDB custom resources from the `mongodb.com` API group.
- **Verification command:**

```bash
kubectl auth can-i list mongodbsearch.mongodb.com \
  -n hyperdx \
  --as=system:serviceaccount:mongodb-operator:mongodb-kubernetes-operator
```

The check returned `yes` after the fix.

> The exact full list of verbs and custom resources in the operator Role should be checked in the live cluster before reproducing it. This README records the permission problem and verified capability; it does not invent a complete Role manifest that was not captured here.

### 2. MongoDB configuration Secret access

- **Service account:** `system:serviceaccount:hyperdx:mongodb-kubernetes-appdb`
- **Role/RoleBinding name:** `hyperdx-mongodb-config-reader`
- **Namespace:** `hyperdx`
- **Resource:** Kubernetes Secret `hyperdx-clickstack-mongodb-config`
- **Purpose:** allow the MongoDB application/agent service account to read the specific configuration Secret needed by the MongoDB pod.
- **Observed issue:** the agent logs reported that it could not get the Secret `hyperdx-clickstack-mongodb-config`.

This permission was scoped to the named Secret rather than granting read access to every Secret in the namespace.

### 3. MongoDB pod access

- **Service account:** `system:serviceaccount:hyperdx:mongodb-kubernetes-appdb`
- **Role/RoleBinding name:** `hyperdx-mongodb-pod-reader`
- **Namespace:** `hyperdx`
- **API group:** core Kubernetes API group (`""`)
- **Resource:** `pods`
- **Final verbs added:** `get`, `patch`

The Role was updated with this command:

```bash
kubectl patch role hyperdx-mongodb-pod-reader -n hyperdx \
  --type=merge \
  -p '{"rules":[{"apiGroups":[""],"resources":["pods"],"verbs":["get","patch"]}]}'
```

The permissions were checked using:

```bash
kubectl auth can-i get pods -n hyperdx \
  --as=system:serviceaccount:hyperdx:mongodb-kubernetes-appdb

kubectl auth can-i patch pods -n hyperdx \
  --as=system:serviceaccount:hyperdx:mongodb-kubernetes-appdb
```

Both checks returned `yes`. The agent subsequently logged that it had reached its goal state, and the MongoDB pod became `2/2 Running`.

### RBAC summary

| Identity | Namespace | Resource / scope | Permission or action |
|---|---|---|---|
| `mongodb-kubernetes-operator` service account | Role/RoleBinding in `hyperdx` | MongoDB custom resources, including `mongodbsearch.mongodb.com` | Operator permissions were added/updated; `list mongodbsearch.mongodb.com` was verified |
| `mongodb-kubernetes-appdb` service account | `hyperdx` | Secret `hyperdx-clickstack-mongodb-config` | Read the named Secret via `hyperdx-mongodb-config-reader` |
| `mongodb-kubernetes-appdb` service account | `hyperdx` | Core resource `pods` | `get`, `patch` via `hyperdx-mongodb-pod-reader` |

### How to inspect the live RBAC configuration

Run these commands to inspect the actual Roles and RoleBindings in the cluster:

```bash
kubectl get role hyperdx-mongodb-config-reader -n hyperdx -o yaml
kubectl get rolebinding hyperdx-mongodb-config-reader -n hyperdx -o yaml

kubectl get role hyperdx-mongodb-pod-reader -n hyperdx -o yaml
kubectl get rolebinding hyperdx-mongodb-pod-reader -n hyperdx -o yaml

kubectl get roles,rolebindings -n hyperdx
```

To inspect the operator Role/RoleBinding, first list the relevant objects and identify their exact names:

```bash
kubectl get roles,rolebindings -n hyperdx
```

**Production note:** these were troubleshooting permissions for the errors observed in this environment. Before production use, compare the live RBAC objects with the documentation for the exact MongoDB Kubernetes Operator version, keep permissions least-privileged, and avoid broad wildcard or cluster-admin grants unless explicitly required and approved.

## 7. Useful operational commands

### Pods, services and Helm

``` bash
kubectl get pods -n hyperdx
kubectl get svc -n hyperdx
kubectl get svc -n clickhouse
helm list -n hyperdx
helm status hyperdx -n hyperdx
```

### Discover deployments and view logs

``` bash
kubectl get deployments -n hyperdx
kubectl logs -n hyperdx deployment/hyperdx-clickstack-app
kubectl logs -n hyperdx deployment/hyperdx-otel-collector
```

If a pod is unhealthy:

``` bash
kubectl describe pod <pod-name> -n hyperdx
kubectl logs <pod-name> -n hyperdx
kubectl logs <pod-name> -n hyperdx -c <container-name> --previous
```

Use `--previous` to inspect the previous crashed container instance. For
multi-container pods, specify the correct container name.

### Inspect MongoDB pod containers

``` bash
kubectl get pod hyperdx-clickstack-mongodb-0 -n hyperdx \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

Then:

``` bash
kubectl logs hyperdx-clickstack-mongodb-0 -n hyperdx -c <container-name>
```

### Open ClickHouse client

First discover the current ClickHouse pod name:

``` bash
kubectl get pods -n clickhouse
```

Then:

``` bash
kubectl exec -it -n clickhouse <clickhouse-pod-name> -- clickhouse-client
```

One-off query example:

``` bash
kubectl exec -n clickhouse <clickhouse-pod-name> -- \
  clickhouse-client --query "SHOW DATABASES"
```

Use the real pod name from the current cluster; do not assume it remains
unchanged.

## 8. Troubleshooting

### HyperDX page does not open

``` bash
kubectl get pods -n hyperdx
kubectl get svc hyperdx-clickstack-app -n hyperdx
```

Ensure port-forward is running in a terminal. If port 3000 is occupied,
forward local port 3001 to service port 3000.

### MongoDB is not ready

``` bash
kubectl describe pod hyperdx-clickstack-mongodb-0 -n hyperdx
kubectl logs hyperdx-clickstack-mongodb-0 -n hyperdx -c <agent-container-name>
```

Use the container discovery command above to get the exact name. Look
for specific RBAC errors and validate permissions with
`kubectl auth can-i`; grant only the missing permission.

### Collector restarts or events are missing

``` bash
kubectl get pods -n hyperdx
kubectl logs -n hyperdx deployment/hyperdx-otel-collector
kubectl logs -n hyperdx deployment/hyperdx-otel-collector --previous
kubectl get svc -n clickhouse
```

Verify the endpoints and credentials in the actual configuration.
Confirm that the telemetry source is configured to send data to the
Collector and that the expected records exist in ClickHouse.

### UI opens but no results appear

-   Check the selected time range.
-   Clear filters and run the query again.
-   Check Collector logs for exporter errors.
-   Confirm the relevant source is instrumented/configured to send
    telemetry.
-   Confirm the intended ClickHouse table/database has records.
-   Confirm HyperDX is querying the intended ClickHouse endpoint.

A healthy UI or successful database connection does not automatically
mean every telemetry source is instrumented or exporting data.

## 9. Security and production considerations

1.  Do not install another ClickHouse unless there is an explicit design
    change.
2.  Do not delete existing ClickHouse PVCs or data directories during
    HyperDX troubleshooting.
3.  Never commit passwords, tokens, or unredacted secrets.
4.  Review the namespace-scoped RBAC added during troubleshooting
    against the installed operator's documented requirements.
5.  Pin Helm chart versions for repeatable releases.
6.  Port-forward is local access. For team-wide/production access, use
    approved ingress, TLS, authentication, and network controls.
7.  Review storage, retention, backup, and resource limits before
    production use.

## 10. Status at last verification

-   ClickStack release deployed in `hyperdx`.
-   HyperDX app pod was `1/1 Running`.
-   MongoDB pod was `2/2 Running`.
-   Collector pod was `1/1 Running`.
-   Collector logs showed a successful ClickHouse connection and
    completed schema/seed operations.
-   HyperDX opened at `http://localhost:3000`.
-   Logs search displayed events, including ClickHouse-related messages.

Recommended follow-up: filter logs by namespace/pod, verify each
intended telemetry source, create a small dashboard, and test an alert
if alerting is in scope.

## 11. Checklist

-   [ ] Confirm Kubernetes context.
-   [ ] Confirm existing ClickHouse health and services.
-   [ ] Confirm MongoDB operator is healthy.
-   [ ] Confirm MongoDB service account and scoped RBAC.
-   [ ] Confirm `clickhouse.enabled: false`.
-   [ ] Confirm both ClickHouse endpoints in the values/config.
-   [ ] Verify Helm release and all HyperDX pods.
-   [ ] Start port-forward and open HyperDX.
-   [ ] Sign in and verify logs.
-   [ ] Filter logs by namespace/pod.
-   [ ] Keep credentials out of Git and review RBAC before production.
