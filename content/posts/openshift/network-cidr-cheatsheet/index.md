---
title: "OpenShift Cluster Network CIDR Planning Cheatsheet"
date: 2026-02-15
categories:
  - Openshift
tags:
  - openshift
  - networking
  - ovn
  - cidr
summary: Comprehensive reference tables for clusterNetwork CIDR, hostPrefix, node count, and pod density combinations
toc:
  enable: true
  auto: true
code:
  maxShownLines: 500
---

Reference tables for planning **clusterNetwork** sizing in OpenShift. The two key parameters are:

- **`clusterNetwork.cidr`** — the overall pod IP space allocated across the cluster
- **`clusterNetwork.hostPrefix`** — the subnet prefix length carved out per node from the cluster CIDR

## How it works

Each node receives a dedicated subnet of size `/hostPrefix` from the `clusterNetwork.cidr` pool. The formulas are:

| Value | Formula |
|-------|---------|
| Max nodes | `2^(hostPrefix − clusterPrefix)` |
| Pod IPs per node | `2^(32 − hostPrefix) − 2` |
| Max routable pod IPs | `Max nodes × Pod IPs per node` |

{{< admonition note >}}
`hostPrefix` must always be **greater than** the cluster CIDR prefix (a larger number = smaller subnet). Values below `/25` per node are unusual in production — Kubernetes itself consumes several IPs per node (kube-proxy, host-network pods, etc.).
{{< /admonition >}}

## Default OpenShift configuration

```yaml
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  networkType: OVNKubernetes
```

This gives: **512 max nodes** × **510 routable IPs/node** = ~261,120 max routable pod IPs across the cluster.

---

## Combination tables

### Cluster CIDR /14  (default — 262,144 total IPs)

{{< admonition note >}}
**Max routable pod IPs** is the total number of IP addresses available for pods across the cluster (`max nodes × pod IPs per node`). It is a hard network ceiling — resource limits (CPU/memory) and Kubernetes internal overhead will exhaust capacity well before this number is reached in practice.
{{< /admonition >}}

| hostPrefix | Max nodes | Pod IPs/node | Max routable pod IPs | Typical use case |
|:----------:|:---------:|:------------:|:--------------------:|------------------|
| /23 | 512 | 510 | 261,120 | **OCP default** — large clusters |
| /24 | 1,024 | 254 | 260,096 | Many nodes, moderate pod density |
| /25 | 2,048 | 126 | 258,048 | Very many nodes, low pod density |
| /26 | 4,096 | 62 | 254,000 | Edge / IoT fleet, minimal pods/node |

### Cluster CIDR /16  (65,536 total IPs)

| hostPrefix | Max nodes | Pod IPs/node | Max routable pod IPs | Typical use case |
|:----------:|:---------:|:------------:|:--------------------:|------------------|
| /23 | 128 | 510 | 65,280 | Medium cluster, high pod density |
| /24 | 256 | 254 | 65,024 | Medium cluster, balanced |
| /25 | 512 | 126 | 64,512 | Many small nodes |
| /26 | 1,024 | 62 | 63,488 | Large node count, minimal pods |

### Cluster CIDR /18  (16,384 total IPs)

| hostPrefix | Max nodes | Pod IPs/node | Max routable pod IPs | Typical use case |
|:----------:|:---------:|:------------:|:--------------------:|------------------|
| /23 | 32 | 510 | 16,320 | Small cluster, high pod density |
| /24 | 64 | 254 | 16,256 | Small cluster, balanced |
| /25 | 128 | 126 | 16,128 | Compact cluster, many nodes |
| /26 | 256 | 62 | 15,872 | Edge cluster, many small nodes |

### Cluster CIDR /20  (4,096 total IPs)

| hostPrefix | Max nodes | Pod IPs/node | Max routable pod IPs | Typical use case |
|:----------:|:---------:|:------------:|:--------------------:|------------------|
| /23 | 8 | 510 | 4,080 | Minimal cluster, dense pods |
| /24 | 16 | 254 | 4,064 | Dev / lab cluster |
| /25 | 32 | 126 | 4,032 | SNO + workers, low density |
| /26 | 64 | 62 | 3,968 | Edge micro-cluster |

### Cluster CIDR /22  (1,024 total IPs)

| hostPrefix | Max nodes | Pod IPs/node | Max routable pod IPs | Typical use case |
|:----------:|:---------:|:------------:|:--------------------:|------------------|
| /24 | 4 | 254 | 1,016 | SNO or 3-node compact |
| /25 | 8 | 126 | 1,008 | Compact + a few workers |
| /26 | 16 | 62 | 992 | Very small lab |

---

## Constraints and rules

| Rule | Detail |
|------|--------|
| `hostPrefix > clusterPrefix` | A node subnet must fit inside the cluster CIDR |
| `hostPrefix ≤ 30` | `/31` and `/32` leave no usable IPs |
| `hostPrefix ≥ 23` recommended | Fewer than 126 pods/node causes scheduling pressure |
| `clusterNetwork` ≠ `serviceNetwork` | The two ranges must not overlap |
| `clusterNetwork` ≠ machine/node CIDRs | Must not overlap with the machine network |

---

## Quick reference: pods per node by hostPrefix

| hostPrefix | Subnet size | Usable pod IPs |
|:----------:|:-----------:|:--------------:|
| /21 | 2,048 | 2,046 |
| /22 | 1,024 | 1,022 |
| /23 | 512 | 510 |
| /24 | 256 | 254 |
| /25 | 128 | 126 |
| /26 | 64 | 62 |
| /27 | 32 | 30 |
| /28 | 16 | 14 |

---

## install-config.yaml snippet

```yaml
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14   # adjust to your sizing
      hostPrefix: 23          # adjust pods/node target
  serviceNetwork:
    - 172.30.0.0/16
  machineNetwork:
    - cidr: 192.168.0.0/24   # must not overlap with clusterNetwork or serviceNetwork
```

{{< admonition warning >}}
`clusterNetwork` and `serviceNetwork` **cannot be changed after installation**. Size conservatively — it is always safe to choose a larger CIDR than currently needed.
{{< /admonition >}}

## Verify on a running cluster

```bash
oc get network cluster -o jsonpath='{.spec.clusterNetwork}' | jq .
```

Expected output:

```json
[
  {
    "cidr": "10.128.0.0/14",
    "hostPrefix": 23
  }
]
```
