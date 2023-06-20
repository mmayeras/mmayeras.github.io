---
title: "Configure OCP network using nmstate operator"
date: 2023-06-20
tags:
- openshift
categories:
- openshift
---

[Official documentation](https://docs.openshift.com/container-platform/4.11/networking/k8s_nmstate/k8s-nmstate-about-the-k8s-nmstate-operator.html)

### Install operator

#### Create NS

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: NMState.v1.nmstate.io
  generateName: openshift-nmstate-
  name: openshift-nmstate-tn6k8
  namespace: openshift-nmstate
spec:
  targetNamespaces:
  - openshift-nmstate
```

#### Create operator group
```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: NMState.v1.nmstate.io
  generateName: openshift-nmstate-
  name: openshift-nmstate-tn6k8
  namespace: openshift-nmstate
spec:
  targetNamespaces:
  - openshift-nmstate
```

#### Create sub
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/kubernetes-nmstate-operator.openshift-nmstate: ""
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

#### Create nmstate instance
```yaml
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
```

### nmstate resources

NodeNetworkState : Current state of nodes
NodeNetworkConfigurationPolicy : Desired state of nodes
NodeNetworkConfigurationEnactment : Reports of the NNCP applied

### NNCP definitions
#### Edit dns search and/or nameservers and add custom routes
In this example, 3 search are added. The primary interface ens18 is used. Initally configured using dhcp static lease I need to configure the same IP adress and ensure default route is created.

> By default, the manifest applies to all nodes in the cluster. To add the interface to specific nodes, add the spec: nodeSelector parameter and the appropriate <key>:<value> for your node selector.
> 
>You can configure multiple nmstate-enabled nodes concurrently. The configuration applies to 50% of the nodes in parallel. This strategy prevents the entire cluster from being unavailable if the network connection fails. To apply the policy configuration in parallel to a specific portion of the cluster, use the maxUnavailable field.

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ens18-custom-dns-and-routes
spec:
  desiredState:
    interfaces:
    - description: custom dns on ens18
      ipv4:
        address:
        - ip: 192.168.0.54
          prefix-length: 24
        auto-dns: false
        auto-gateway: false
        auto-routes: false
        dhcp: false
        enabled: true
      name: ens18
      state: up
      type: ethernet
    routes:
      config:
      - destination: 0.0.0.0/0
        metric: 150
        next-hop-address: 192.168.0.1
        next-hop-interface: ens18
        table-id: 254
      - destination: 172.16.0.0/12
        metric: 150
        next-hop-address: 192.168.0.10
        next-hop-interface: ens18
        table-id: 254
    dns-resolver:
      config:
        search:
        - my-extra-dns1
        - my-extra-dns2
        - my-extra-dns3
        server:
        - 8.8.8.8
  nodeSelector:
    kubernetes.io/hostname: master3
```

### Once applied, wait until status Available

```bash
$ oc get nncp                                              
NAME                            STATUS      REASON
ens18-custom-dns-and-routes     Available   SuccessfullyConfigur
```

**If status is degraded, use the following command to get message from nnce**

```bash
$ oc get nnce <nnce_name> -o jsonpath='{.status.conditions[?(@.type=="Failing")].message}'                                            
```

Here is an examplemessage example when using an unreachable nameserver :

```
error reconciling NodeNetworkConfigurationPolicy at desired state apply: ,

  failed checking DNS connectivity
    [lookup root-server.net on 192.168.0.10:53
      read udp 192.168.0.54:49374->1.2.3.4:53
        i/o timeout]
```

**Fix the nncp depending on the error message**





