+++
title = "External ODF POC"
date = 2023-02-15T09:39:20+01:00
weight = 30
chapter = true
+++

### In this section, a fresh 4.10 cluster will use ODF to connect to an external RHCS 5.3 cluster

The installation of the cluster is not covered here, refer to the installation section.
The install-config.yaml for reference is the following :

```yaml
apiVersion: v1
baseDomain: CHANGEME
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: CHANGEME
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: CHANGEME
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  vsphere:
    apiVIP: CHANGEME
    cluster: CHANGEME
    datacenter: CHANGEME
    defaultDatastore: CHANGEME
    ingressVIP: CHANGEME
    network: CHANGEME
    password: CHANGEME
    username: CHANGEME
    vCenter: CHANGEME
publish: External
pullSecret: 'CHANGEME'
sshKey: |
  ssh-rsa CHANGEME
```
