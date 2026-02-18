---
title: "Enable LLDP on OpenShift nodes using nmstate"
date: 2025-02-18
draft: false
categories:
- openshift
tags:
- openshift
- nmstate
- lldp
---

This article describes how to enable [LLDP](https://en.wikipedia.org/wiki/Link_Layer_Discovery_Protocol) (Link Layer Discovery Protocol) on all Ethernet interfaces that are up, using the [nmstate operator](https://docs.openshift.com/container-platform/latest/networking/k8s_nmstate/k8s-nmstate-about-the-k8s-nmstate-operator.html) and a **NodeNetworkConfigurationPolicy** (NNCP).

Prerequisites: the Kubernetes NMState operator is installed and a `NMState` instance exists (see [Configure OCP network using nmstate operator](/posts/openshift/nmstate/)).

## 1. Apply the NodeNetworkConfigurationPolicy

The policy below uses nmstate **capture** to select only Ethernet interfaces that are in state `up`, then sets `lldp.enabled: true` on those interfaces. This avoids touching other interface types or down interfaces.

```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: enable-lldp-ethernets-up
spec:
  capture:
    ethernets: interfaces.type=="ethernet"
    ethernets-up: capture.ethernets | interfaces.state=="up"
    ethernets-up-lldp: capture.ethernets-up | interfaces.lldp.enabled:=true
  desiredState:
    interfaces: '{{ capture.ethernets-up-lldp.interfaces }}'
EOF
```

Wait until the policy is applied on all targeted nodes:

```bash
oc get nncp
oc get nnce   # optional: check enactments per node
```

## 2. Verify LLDP is enabled

To confirm that LLDP is enabled on Ethernet interfaces that are up, use **NodeNetworkState** (NNS) and filter with `jq`:

```bash
oc get nns -o json | jq '.items[] | .metadata.name as $node | .status.currentState.interfaces[] | select(.type=="ethernet" and .state=="up") | {node: $node, iface: .name, lldp: .lldp.enabled}'
```

Example output:

```json
{"node": "master-0", "iface": "ens18", "lldp": true}
{"node": "worker-0", "iface": "ens18", "lldp": true}
```

If `lldp` is `true` for each up Ethernet interface, LLDP is correctly enabled via nmstate.
