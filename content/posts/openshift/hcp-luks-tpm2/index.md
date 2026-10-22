---
title: "Hosted control planes: TPM2-bound LUKS on agent workers"
date: 2026-10-22
draft: false
categories:
  - Openshift
tags:
  - openshift
  - hypershift
  - mco
  - security
  - kubevirt
  - nmstate
---

Lab walkthrough: a Hosted Control Planes (HCP) cluster on the Agent platform with a worker VM on OpenShift Virtualization (persistent vTPM, static IPv4). Two independent customizations are applied from the hub, each in its own **NodePool** config ConfigMap:

- **LUKS disk encryption**: the worker root disk is encrypted with LUKS and unlocked by the TPM 2.0.
- **`core` password**: a console password for the `core` user, on the installed node and, separately, on the discovery image.

Either one can be used without the other.

<!--more-->

*Acronyms: HCP = Hosted Control Planes, LUKS = Linux Unified Key Setup, TPM = Trusted Platform Module, MCE = multicluster engine for Kubernetes, CAPI = Cluster API.*

{{< admonition info "Lab repository: github.com/mmayeras/hcp_luks" >}}
Every manifest below lives in **[https://github.com/mmayeras/hcp_luks](https://github.com/mmayeras/hcp_luks)**, with placeholders for all environment values:

- [`manifests/`](https://github.com/mmayeras/hcp_luks/tree/main/manifests): HostedCluster, NodePool, NodePool config ConfigMaps, InfraEnv, NMStateConfig, agent Role.
- [`vm/`](https://github.com/mmayeras/hcp_luks/tree/main/vm): worker VM and discovery ISO DataVolume.
- [`butane/`](https://github.com/mmayeras/hcp_luks/tree/main/butane): Butane sources of both MachineConfigs.
- [`README.md`](https://github.com/mmayeras/hcp_luks/blob/main/README.md): placeholder table and the full apply order.
{{< /admonition >}}

HCP is built on the upstream [HyperShift](https://github.com/openshift/hypershift) project, the Agent platform on [assisted-service](https://github.com/openshift/assisted-service), disk encryption on [Ignition](https://github.com/coreos/ignition) and [Clevis](https://github.com/latchset/clevis), and the worker VM on [KubeVirt](https://github.com/kubevirt/kubevirt).

## Topology

```text
  Hub (OpenShift 4.22, MCE 2.17, OpenShift Virtualization, ODF)
  +--------------------------------------------------------------------+
  | ns clusters            HostedCluster hcp-luks, NodePool,           |
  |                        ConfigMap hcp-luks-worker-luks-tpm2         |
  |                        ConfigMap hcp-luks-worker-core-password     |
  | ns clusters-hcp-luks   hosted control plane pods + ignition server |
  | ns <AGENT_NAMESPACE>   InfraEnv, NMStateConfig, Agent,             |
  |                        VM hcp-luks-worker-0 (persistent vTPM)      |
  +--------------------------------------------------------------------+
            discovery ISO -> Agent -> install -> first boot from disk:
            Ignition encrypts root, seals the key in the vTPM
```

Key point: the NodePool configuration is rendered by the HCP ignition server and applied by Ignition on the **first boot from disk**. The discovery image never shows it.

## Prerequisites

- MCE with HyperShift and the Agent (assisted installer) components enabled on the hub.
- OpenShift Virtualization installed, with `spec.vmStateStorageClass` set in the **HyperConverged** CR (filesystem volume mode, required by the persistent vTPM).
- A LoadBalancer provider (MetalLB) for the hosted API server.
- A NetworkAttachmentDefinition on the machine network (bridge or OVN localnet).
- `oc`, `virtctl`, `butane`, `jq` available in `$PATH`.

The commands below use the agent namespace (InfraEnv, Agent, VM) from a variable:

```bash
AGENT_NS=<AGENT_NAMESPACE>
```

## 1. LUKS: encrypt the root disk with the TPM

Butane translates `boot_device.luks.tpm2` into Ignition `storage.luks` and `storage.filesystems` entries:

```bash
cat > worker-luks-tpm2.bu <<EOF
variant: openshift
version: 4.18.0
metadata:
  name: 99-worker-luks-tpm2
  labels:
    machineconfiguration.openshift.io/role: worker
boot_device:
  layout: x86_64
  luks:
    tpm2: true
EOF
butane worker-luks-tpm2.bu
```

Butane 0.23 stops at variant `openshift 4.18.0`. The output uses Ignition `3.4.0`, which newer releases accept. A NodePool reads MachineConfigs from a ConfigMap key named `config`, in the NodePool namespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hcp-luks-worker-luks-tpm2
  namespace: clusters
data:
  config: |
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      name: 99-worker-luks-tpm2
      labels:
        machineconfiguration.openshift.io/role: worker
    spec:
      config:
        ignition:
          version: 3.4.0
        storage:
          luks:
            - name: root
              label: luks-root
              device: /dev/disk/by-partlabel/root
              clevis:
                tpm2: true
              wipeVolume: true
          filesystems:
            - device: /dev/mapper/root
              format: xfs
              label: root
              wipeFilesystem: true
```

`tpm2: true` seals the key to the TPM without PCR policy. To also bind it to the Secure Boot state, replace the `clevis` block with `custom: {pin: tpm2, config: '{"pcr_bank":"sha256","pcr_ids":"7"}'}`. A firmware or Secure Boot change then stops the automatic unlock.

## 2. Core password: console login on the worker

This item is independent of LUKS. The password is set in two separate places, because the discovery image and the installed node get their Ignition from different sources.

### Installed node: NodePool MachineConfig

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hcp-luks-worker-core-password
  namespace: clusters
data:
  config: |
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      name: 99-worker-core-password
      labels:
        machineconfiguration.openshift.io/role: worker
    spec:
      config:
        ignition:
          version: 3.4.0
        passwd:
          users:
            - name: core
              passwordHash: '<CORE_PASSWORD_HASH>'
```

Generate the hash with `openssl passwd -6`. The password works on the console only; RHCOS keeps SSH password authentication disabled.

### Discovery image: InfraEnv ignitionConfigOverride

The NodePool configuration never reaches the discovery image. A console password there helps when a wrong static IP leaves the host unreachable. The **InfraEnv** field `ignitionConfigOverride` is merged into the discovery Ignition. Once the InfraEnv from step 4 exists:

```bash
HASH='<CORE_PASSWORD_HASH>'
IGN=$(jq -cn --arg h "$HASH" '{ignition:{version:"3.1.0"},passwd:{users:[{name:"core",passwordHash:$h}]}}')
oc -n "$AGENT_NS" patch infraenv hcp-luks --type merge \
  -p "$(jq -cn --arg i "$IGN" '{spec:{ignitionConfigOverride:$i}}')"
```

The repository sets the same field directly in [`02-infraenv.yaml`](https://github.com/mmayeras/hcp_luks/blob/main/manifests/02-infraenv.yaml). Any change regenerates the ISO under a new URL.

## 3. Reference both configs from the NodePool

Each item is a separate entry in `spec.config`; removing one entry drops that item only.

```yaml
apiVersion: hypershift.openshift.io/v1beta1
kind: NodePool
metadata:
  name: hcp-luks-workers
  namespace: clusters
spec:
  clusterName: hcp-luks
  replicas: 1
  arch: amd64
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.22.15-x86_64
  management:
    autoRepair: false
    upgradeType: Replace
    replace:
      strategy: RollingUpdate
      rollingUpdate:
        maxSurge: 0
        maxUnavailable: 1
  platform:
    type: Agent
    agent:
      agentLabelSelector:
        matchLabels:
          hcp-luks.example.com/tpm2: "true"
  config:
    - name: hcp-luks-worker-luks-tpm2
    - name: hcp-luks-worker-core-password
```

{{< admonition warning "Replace, not InPlace" >}}
Ignition encrypts the disk at first boot only. An `InPlace` NodePool cannot encrypt an existing root disk. With `Replace`, any change to a referenced ConfigMap (the password included) reprovisions every node of the pool. With a single worker and no spare Agent, the node is removed first.
{{< /admonition >}}

## 4. Prepare the agent namespace

The **InfraEnv** generates the discovery ISO; `nmStateConfigLabelSelector` embeds the static network configuration. The discovery `core` password from step 2 is added to it with `ignitionConfigOverride`:

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: hcp-luks
  namespace: <AGENT_NAMESPACE>
spec:
  pullSecretRef:
    name: hcp-luks-pull-secret
  sshAuthorizedKey: '<SSH_PUBLIC_KEY>'
  cpuArchitecture: x86_64
  agentLabels:
    hcp-luks.example.com/tpm2: "true"
  nmStateConfigLabelSelector:
    matchLabels:
      infraenvs.agent-install.openshift.io: hcp-luks
```

The **NMStateConfig** is matched to the host by NIC MAC address; `eno1` is only a logical name:

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: hcp-luks-worker-0
  namespace: <AGENT_NAMESPACE>
  labels:
    infraenvs.agent-install.openshift.io: hcp-luks
spec:
  interfaces:
    - name: eno1
      macAddress: '<WORKER0_MAC>'
  config:
    interfaces:
      - name: eno1
        type: ethernet
        state: up
        mac-address: '<WORKER0_MAC>'
        ipv4:
          enabled: true
          dhcp: false
          address:
            - ip: 192.0.2.50
              prefix-length: 24
        ipv6:
          enabled: false
    dns-resolver:
      config:
        server:
          - 192.0.2.53
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 192.0.2.1
          next-hop-interface: eno1
          table-id: 254
```

The agent namespace also needs the pull secret (a second copy, the HostedCluster reads its own in `clusters`) and the Role used by the Agent CAPI provider:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: capi-provider-role
  namespace: <AGENT_NAMESPACE>
rules:
  - apiGroups:
      - agent-install.openshift.io
    resources:
      - agents
    verbs:
      - '*'
```

{{< admonition warning "capi-provider-role is not created by the operator" >}}
`hcp create cluster agent --render` emits this Role; the HyperShift operator only creates the RoleBinding to it. Without the Role, the HostedCluster loops on `failed to reconcile Agent RoleBinding: rolebindings.rbac.authorization.k8s.io "capi-provider-role" not found` and the control plane namespace stays empty (observed on MCE 2.17.3).
{{< /admonition >}}

## 5. Create the HostedCluster

```yaml
apiVersion: hypershift.openshift.io/v1beta1
kind: HostedCluster
metadata:
  name: hcp-luks
  namespace: clusters
spec:
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.22.15-x86_64
  pullSecret:
    name: hcp-luks-pull-secret
  sshKey:
    name: hcp-luks-ssh-key
  infraID: hcp-luks
  dns:
    baseDomain: <BASE_DOMAIN>
  controllerAvailabilityPolicy: HighlyAvailable
  infrastructureAvailabilityPolicy: SingleReplica
  olmCatalogPlacement: management
  fips: false
  networking:
    networkType: OVNKubernetes
    clusterNetwork:
      - cidr: 10.132.0.0/14
    serviceNetwork:
      - cidr: 172.31.0.0/16
    machineNetwork:
      - cidr: 192.0.2.0/24
  platform:
    type: Agent
    agent:
      agentNamespace: <AGENT_NAMESPACE>
  etcd:
    managementType: Managed
    managed:
      storage:
        type: PersistentVolume
        persistentVolume:
          size: 8Gi
          storageClassName: ocs-storagecluster-ceph-rbd
  services:
    - service: APIServer
      servicePublishingStrategy:
        type: LoadBalancer
    - service: OAuthServer
      servicePublishingStrategy:
        type: Route
    - service: Konnectivity
      servicePublishingStrategy:
        type: Route
    - service: Ignition
      servicePublishingStrategy:
        type: Route
```

`infrastructureAvailabilityPolicy: SingleReplica` runs guest infrastructure components (ingress router) with one replica, matching the single worker.

## 6. Create the worker VM

The MAC address is not known before the VM exists, while the ISO needs it in the NMStateConfig. The VM is therefore created `Halted` and without `macAddress`: kubemacpool assigns one and writes it into the VM spec.

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: hcp-luks-worker-0
  namespace: <AGENT_NAMESPACE>
spec:
  runStrategy: Halted
  dataVolumeTemplates:
    - metadata:
        name: hcp-luks-worker-0-root
      spec:
        source:
          blank: {}
        storage:
          accessModes:
            - ReadWriteMany
          volumeMode: Block
          resources:
            requests:
              storage: 120Gi
          storageClassName: ocs-storagecluster-ceph-rbd-virtualization
  template:
    metadata:
      labels:
        kubevirt.io/domain: hcp-luks-worker-0
    spec:
      domain:
        cpu:
          cores: 8
        memory:
          guest: 16Gi
        features:
          smm:
            enabled: true
        firmware:
          bootloader:
            efi:
              secureBoot: true
              persistent: true
        devices:
          tpm:
            persistent: true
          disks:
            - name: rootdisk
              bootOrder: 1
              disk:
                bus: virtio
            - name: discovery-iso
              bootOrder: 2
              cdrom:
                bus: sata
          interfaces:
            - name: nic0
              model: virtio
              bridge: {}
      networks:
        - name: nic0
          multus:
            networkName: <NAD_NAMESPACE>/<NAD_NAME>
      volumes:
        - name: rootdisk
          dataVolume:
            name: hcp-luks-worker-0-root
        - name: discovery-iso
          dataVolume:
            name: hcp-luks-discovery-iso
```

{{< admonition warning "Persistent vTPM and boot order" >}}
Without `tpm.persistent: true`, the vTPM state is lost on VM restart and the sealed LUKS key with it: the root disk never unlocks again. Keep `rootdisk` first in the boot order: the blank disk falls through to the ISO, and after installation the VM boots the encrypted disk instead of the discovery image.
{{< /admonition >}}

## 7. Build the ISO, boot, approve

```bash
oc apply -f vm/02-worker-vm.yaml
oc -n "$AGENT_NS" get vm hcp-luks-worker-0 \
  -o jsonpath='{.spec.template.spec.domain.devices.interfaces[0].macAddress}{"\n"}'
# copy the MAC into 03-nmstateconfig.yaml, then
oc apply -k manifests
oc -n "$AGENT_NS" wait infraenv/hcp-luks --for=condition=ImageCreated --timeout=5m
ISO_URL=$(oc -n "$AGENT_NS" get infraenv hcp-luks -o jsonpath='{.status.isoDownloadURL}')
sed "s|<ISO_URL>|${ISO_URL}|" vm/01-iso-datavolume.yaml | oc apply -f -
oc -n "$AGENT_NS" wait dv/hcp-luks-discovery-iso --for=condition=Ready --timeout=15m
virtctl -n "$AGENT_NS" start hcp-luks-worker-0
oc -n "$AGENT_NS" get agents
oc -n "$AGENT_NS" patch agent <AGENT_ID> --type merge -p '{"spec":{"approved":true}}'
```

The ISO URL carries a token: it is injected at apply time and never stored in the repository. Any change to the InfraEnv or a matching NMStateConfig regenerates the ISO under a new URL.

## Verify

Discovery stage, Agent registered and waiting for the NodePool:

```bash
oc -n "$AGENT_NS" get agents \
  -o custom-columns=HOST:.spec.hostname,APPROVED:.spec.approved,STATE:.status.debugInfo.state
```

```text
HOST                APPROVED   STATE
hcp-luks-worker-0   true       known-unbound
```

The discovery image carries neither the LUKS configuration nor the NodePool password: its root disk is still blank. Both NodePool items apply on the first boot from disk after installation.

### LUKS

```bash
lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINT
sudo cryptsetup status root
sudo clevis luks list -d /dev/vda4
```

```text
[core@hcp-luks-worker-0 ~]$ lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINT
sudo cryptsetup status root
sudo clevis luks list -d /dev/vda4
NAME     TYPE  FSTYPE      MOUNTPOINT
loop0    loop  erofs
sr0      rom   iso9660
vda      disk
├─vda1   part
├─vda2   part  vfat
├─vda3   part  ext4        /boot
└─vda4   part  crypto_LUKS
  └─root crypt xfs         /sysroot
/dev/mapper/root is active and is in use.
  type:    LUKS2
  cipher:  aes-xts-plain64
  keysize: 512 [bits]
  key location: keyring
  device:  /dev/vda4
  sector size:  512 [bytes]
  offset:  32768 [512-byte units] (16777216 [bytes])
  size:    250574815 [512-byte units] (128294305280 [bytes])
  mode:    read/write
1: sss '{"t":1,"pins":{"tpm2":[{"hash":"sha256","key":"ecc"}]}}'
```

### Core password

- Discovery image: `virtctl -n "$AGENT_NS" console hcp-luks-worker-0`, log in as `core` with the `ignitionConfigOverride` password.
- Installed node:

```bash
sudo grep '^core:' /etc/shadow | cut -c1-8
```

```text
[core@hcp-luks-worker-0 ~]$ sudo grep '^core:' /etc/shadow | cut -c1-8
core:$6$
```
