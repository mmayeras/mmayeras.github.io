---
title: "Installer-provisioned installation"
date: 2023-03-24
tags:
- openshift
categories:
- openshift
# resources:
# - name: "featured-image"
#   src: "ocp.png"

---


### IPI

1. Get openshift installer, openshift cli and pull-secret from https://console.redhat.com/openshift

2. Create install-config.yaml
`$ openshift-install create install-config --dir ./cluster`

Here is a sample install-config.yaml for vSphere IPI
```yaml
additionalTrustBundlePolicy: Proxyonly
apiVersion: v1
baseDomain: example.com
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
  name: mmayeras
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.10.0.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  vsphere:
    apiVIPs:
    - 10.10.0.2
    cluster: your_cluster
    datacenter: your_datacenter
    defaultDatastore: your_datastore
    ingressVIPs:
    - 10.10.0.3
    network: your_network
    password: your_password
    username:  your_username
    vCenter: your_vcenter
publish: External
pullSecret: 'your_pull_secret'
sshKey: your_ssh_pub_key
```

3. Backup and Copy the install-config.yaml into the installation dir
`$ cp install-config.yaml{,.bak} && mv install-config.yaml ./cluster/`
4.  Launch the installer
`$ openshift-install create cluster --dir ./cluster`