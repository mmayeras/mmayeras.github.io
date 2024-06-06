---
title: "Configure OCP network using nmstate operator"
date: 2023-06-20
tags:
- openshift
categories:
- openshift
---

[Official documentation](https://docs.openshift.com/container-platform/4.11/networking/k8s_nmstate/k8s-nmstate-about-the-k8s-nmstate-operator.html)

## Install operator

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

### Create nmstate instance
```yaml
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
```

## nmstate resources

**NodeNetworkState** : Current state of nodes

**NodeNetworkConfigurationPolicy** : Desired state of nodes

**NodeNetworkConfigurationEnactment** : Reports of the NNCP applied

## NNCP definition

### **Edit dns search and/or nameservers and add custom routes**

In this example, 3 search are added. The primary interface ens18 is used. Initally configured using dhcp static lease I need to configure the same IP address and ensure default route is created.

{{< admonition danger "Danger" >}}
By default, the manifest applies to all nodes in the cluster. To add the interface to specific nodes, add the spec: nodeSelector parameter and the appropriate <key>:<value> for your node selector.

You can configure multiple nmstate-enabled nodes concurrently. The configuration applies to 50% of the nodes in parallel. This strategy prevents the entire cluster from being unavailable if the network connection fails. To apply the policy configuration in parallel to a specific portion of the cluster, use the maxUnavailable field.
{{< /admonition >}}


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

Here is an example message when using an unreachable nameserver :

```
error reconciling NodeNetworkConfigurationPolicy at desired state apply: ,

  failed checking DNS connectivity
    [lookup root-server.net on 192.168.0.10:53
      read udp 192.168.0.54:49374->1.2.3.4:53
        i/o timeout]
```

  {{< admonition warning "Warning" >}}
    Fix the nncp depending on the error message
  {{< /admonition >}}


### NetworkManager config file result on the node 


```ini
# /etc/NetworkManager/system-connections/ens18.nmconnection
[connection]
id=ens18
uuid=xxxxxxxxxxxxxxxxxxxxxxx
type=ethernet
interface-name=ens18
lldp=0

[ethernet]

[ipv4]
address1=192.168.0.54/24
dhcp-client-id=mac
dhcp-timeout=90
dns=8.8.8.8;
dns-priority=40
dns-search=my-extra-dns1;my-extra-dns2;my-extra-dns3;
may-fail=false
method=manual
route1=172.16.0.0/12,192.168.0.10,150
route1_options=table=254
route2=0.0.0.0/0,192.168.0.1,150
route2_options=table=254

[user]
nmstate.interface.description=custom dns on ens18 2
```




