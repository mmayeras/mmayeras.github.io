---
title: "Monitor RHOAI usage with Cluster Observability Operator and Perses"
date: 2026-04-21
categories:
  - Openshift
tags:
  - openshift
  - rhoai
  - monitoring
  - perses
  - coo
  - vllm
  - gpu
---

Deploy the [Cluster Observability Operator](https://github.com/rhobs/cluster-observability-operator) with its Perses UI plugin, then apply a **PersesDashboard** that surfaces Red Hat OpenShift AI workbench, pipeline server, and model-serving metrics directly in the OpenShift web console.

<!--more-->

## Prerequisites

- **Red Hat OpenShift AI** (RHOAI) is installed and at least one data science project exists (see [RHOAI docs](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/latest/html/installing_and_uninstalling_openshift_ai_self-managed/index)).
- `oc` available in `$PATH` with `cluster-admin` privileges.

## 1. Enable User Workload Monitoring

User Workload Monitoring (UWM) is required so RHOAI components can expose metrics to the `openshift-user-workload-monitoring` Prometheus.

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

Wait until the UWM stack is ready:

```bash
oc rollout status statefulset/prometheus-user-workload -n openshift-user-workload-monitoring
```

## 2. Install the Cluster Observability Operator

Create a **Subscription** in `openshift-operators` to pull COO from the `redhat-operators` catalog:

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-operators
spec:
  channel: development <!-- VERIFY: confirm channel name in OperatorHub -->
  installPlanApproval: Automatic
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Confirm the operator pod reaches `Running`:

```bash
oc get pods -n openshift-operators -l app.kubernetes.io/name=observability-operator
```

## 3. Enable the Perses UI plugin

The **UIPlugin** resource instructs COO to deploy the Perses console plugin. Once active, `PersesDashboard` objects become navigable from the OpenShift web console under **Observe → Dashboards**.

```bash
cat <<EOF | oc apply -f -
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  name: perses
spec:
  type: Dashboards
EOF
```

{{< admonition tip "Datasource strategy" >}}
The dashboard's `$namespace` variable queries `kube_namespace_labels` with `label_opendatahub_io_dashboard="true"` to enumerate ODH-owned projects. That metric is only available on the platform Prometheus (`openshift-monitoring`). If Perses is configured with a single UWM datasource, switch to the fallback matcher `kube_pod_labels{label_notebook_name!=""}` defined in the dashboard's variable comments.
{{< /admonition >}}

Wait for the plugin pod:

```bash
oc rollout status deployment/perses -n openshift-operators
```

## 4. Apply the PersesDashboard

The manifest creates a six-section dashboard in the `redhat-ods-monitoring` namespace (created by RHOAI at install time). The sections are:

| Section | Panels |
|---------|--------|
| Overview | System health, deployed models, GPU utilization, request success rate |
| Workbench & pipeline activity | Running workbenches, pipeline servers, active pipeline runs |
| Cluster resource overview | GPU / memory / CPU / inbound network time-series |
| Workbench resource detail | CPU and memory per workbench (filtered by `$notebook`) |
| Pipeline server resource detail | CPU, memory, and run activity per pipeline server |
| Project resource usage | GPU, CPU, memory stacked by project |

```bash
oc apply -f dashboard.yaml
```

The three template variables exposed at the top of the dashboard are:

| Variable | Source metric | Purpose |
|----------|--------------|---------|
| `$namespace` | `kube_namespace_labels{label_opendatahub_io_dashboard="true"}` | Filter to ODH projects |
| `$notebook` | `kube_pod_labels{label_notebook_name!=""}` | Filter to specific workbenches |
| `$pipeline_server` | `kube_pod_labels{label_dspa!=""}` | Filter to specific DSPA instances |

## Verify

Confirm the **PersesDashboard** object is accepted:

```bash
oc get persesdashboard -n redhat-ods-monitoring
```

Expected output:

```
NAME                      AGE
dashboard-rhoai-filters   30s
```

Check that the UIPlugin reports a healthy status:

```bash
oc get uiplugin perses -o jsonpath='{.status.conditions}' | jq .
```

```json
[
  {
    "lastTransitionTime": "2026-04-21T10:00:00Z",
    "message": "UIPlugin is available",
    "reason": "Available",
    "status": "True",
    "type": "Available"
  }
]
```

Navigate to **Observe → Dashboards** in the OpenShift web console and select **Cluster Details per project** from the dropdown to access the dashboard.

## Screenshots

Workbench & pipeline activity, cluster resource overview:

![Dashboard overview](dashboard-overview.png)

Workbench resource detail — CPU and memory per workbench pod:

![Workbench resource detail](dashboard-workbench-detail.png)

Project resource usage — GPU, CPU, and memory stacked by namespace:

![Project resource usage](dashboard-project-usage.png)
