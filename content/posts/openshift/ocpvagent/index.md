+++
title = "Single Node Openshift running on OCP-V disconnected"
date = 2025-03-03
+++

#### LAB overview


In this LAB, I'm going to deploy a single node OpenShift Cluster in Openshift Virtualization using a private network without internet access.


#### 1. Requirements


- A linux machine with clients installed
    - Openshift Client https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz
    - Openshift install client https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.17.16/openshift-install-linux.tar.gz
    - oc mirror https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/oc-mirror.rhel9.tar.gz
    - virtctl (Get the downlond links from the Openshift Console)

- A mirror registry to store Openshift Images
Mirror registry for Red Hat openshift will be used in this lab

The mirror registry for Red Hat OpenShift is a small and streamlined container registry that you can use as a target for mirroring the required container images of OpenShift Container Platform for disconnected installations.

https://docs.openshift.com/container-platform/4.17/disconnected/mirroring/installing-mirroring-creating-registry.html

https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz


- A valid pull secret 

- A running Openshift Cluster with Openshift Virtualization up and running


#### 2. Deploy the registry

```shell
$ tar xzf mirror-registry-amd64.tar.gz
$ ./mirror-registry install --quayHostname <vm_hostname> --quayRoot /mirror417/ --initUser foo --initPassword password --sslCheckSkip
````

Once installation succeeded, verify the instance is up

```shell
$ curl -k https://<vm_hostname>:8443/health/instance
{"data":{"services":{"auth":true,"database":true,"disk_space":true,"registry_gunicorn":true,"service_key":true,"web_gunicorn":true}},"status_code":200}
````

And confirm you can login to the registry

```shell
$ podman login <vm_hostname>:8443 -u foo -p password
```
#### 3. Mirror images for a disconnected installation

https://docs.openshift.com/container-platform/4.17/disconnected/mirroring/about-installing-oc-mirror-v2.html

Download your registry.redhat.io pull secret from [Red Hat OpenShift Cluster Manager](https://console.redhat.com/openshift/install/pull-secret).

```shell
$ cat ./pull-secret | jq . > pull-secret.json
```

Add an entry for you private registry 

$ echo -n 'foo:password' | base64 -w0
BGVtbYk3ZHAtqXs=

```json
  "auths": {
    "<vm_hostname>:8443": { 
      "auth": "<credentials>", 
      "email": "you@example.com"
    }
  },
``` 

Save the pull secret file as $XDG_RUNTIME_DIR/containers/auth.json.

Create an image set configuration

{{< admonition info >}}
  The version mirrored must match the version of the openshift-install client
{{< /admonition >}}

```yaml
#imageset.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  platform:
    channels:
    - name: stable-4.17
      minVersion: 4.17.16
      maxVersion: 4.17.16
    graph: true
```

Mirror to disk first

```shell
$ oc mirror -c imageset.yaml file:///mirror417/ --v2
```
Then transfer the container images to the registry
```shell
$ oc mirror -c imageset.yaml --from file:///mirror417/ docker:/<vm_hostname>:8443 --v2
```
Ensure the IDMS is created in /mirror417/working-dir/cluster-resources/idms-oc-mirror.yaml

```yaml
kind: ImageDigestMirrorSet
metadata:
  name: idms-release-0
spec:
  imageDigestMirrors:
  - mirrors:
    - <vm_hostname>:8443/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - <vm_hostname>:8443/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
status: {}
```


#### 4. Create the cluster installation files
##### Option 1 : Boot from ISO

Edit the following files with your values

```yaml
#install-config.yaml
apiVersion: v1
baseDomain: test.mmayeras.local.ocpv
additionalTrustBundle: | #Get the CA from your mirror regsitry installation dir e.g. /mirror417/quay-rootCA/rootCA.pem
  -----BEGIN CERTIFICATE-----
  
  -----END CERTIFICATE-----
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 1
metadata:
  name: sno-cluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.100.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '' #Paste your pull secret here
sshKey: '' #Paste an SSH public key here
imageContentSources: #Paste the mirrors from the idms-oc-mirror.yaml file here
- mirrors:
    - <vm_hostname>:8443/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
    - <vm_hostname>:8443/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
```


```yaml
#agent-config.yaml

apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: sno-cluster
rendezvousIP: 192.168.100.10
hosts:
  - hostname: master-0
    interfaces:
      - name: enp1s0
        macAddress: 02:a1:3b:00:00:56
    rootDeviceHints:
      deviceName: /dev/vda
    networkConfig:
      interfaces:
        - name: enp1s0
          type: ethernet
          state: up
          mac-address: 02:a1:3b:00:00:56 
          ipv4:
            enabled: true
            address:
              - ip: 192.168.100.10
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.100.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.100.1
            next-hop-interface: enp1s0
            table-id: 254

```


Create the bootable ISO

```shell
$ mkdir install_dir
$ cp install-config.yaml agent-config.yaml install_dir/
$ ./openshift-install agent create image --dir install_dir/
```

##### Option 2 : PXE Boot

dnsmasq configuration for ipxe

```
user=dnsmasq
group=dnsmasq
interface=eth1
interface=lo
bind-interfaces
addn-hosts=/etc/mmayeras.local.ocpv.hosts
expand-hosts
domain=mmayeras.local.ocpv
dhcp-range=192.168.100.100,192.168.100.150,255.255.255.0,12h
dhcp-option=3
dhcp-option=option:dns-server,192.168.100.1
dhcp-boot=agent.x86_64.ipxe
pxe-prompt="PXE Booting in", 5
pxe-service=X86PC, "Boot from network", agent.x86_64.ipxe
enable-tftp
tftp-root=/var/lib/tftpboot
conf-dir=/etc/dnsmasq.d,.rpmnew,.rpmsave,.rpmorig
```

Add bootArtifactsBaseURL in the agent-config.yaml

bootArtifactsBaseURL: http://192.168.100.1/

Create the PXE files

```shell
$ mkdir install_dir
$ cp install-config.yaml agent-config.yaml install_dir/
$ ./openshift-install agent create pxe-files --dir install_dir/
```

Copy the ipxe file in /var/lib/tftpboot and initrd, rootfs, vmlinuz in /var/www/html/



#### 5. Build the Openshift Cluster


{{< admonition warning >}}
  A DNS resolver must exist in the private network to allow name resolution of the container registry hostname 
{{< /admonition >}}



Create a bootable datavolume
```shell
$ oc new-project mysno
$ ./virtctl image-upload dv <my-dv> --size=2G --image-path=install_dir/agent.x86_64.iso --insecure
````

NNCP and NAD specs

```yaml
#NNCP Private NIC
spec:
  desiredState:
    interfaces:
    - bridge:
        options:
          stp:
            enabled: false
        port:
        - name: ens1f1
      description: Linux bridge with ens1f1 as a port
      name: br1 #This is the private NIC 
      state: up
      type: linux-bridge


# NAD 
spec:
  config: |-
    {
        "cniVersion": "0.3.1",
        "name": "br1-private",
        "type": "bridge",
        "bridge": "br1",
        "ipam": {},
        "macspoofchk": false,
        "preserveDefaultVlan": false
    }
```


Create a new VM

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:  
  name: my-sno-vm
  namespace: mysno
spec:
  dataVolumeTemplates:
  - metadata:
      creationTimestamp: null
      name:  my-sno-vm-volume-blank
    spec:
      source:
        blank: {}
      storage:
        resources:
          requests:
            storage: 150Gi
        storageClassName: ocs-storagecluster-ceph-rbd-virtualization
  instancetype:
    kind: VirtualMachineClusterInstancetype
    name: u1.2xlarge
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        network.kubevirt.io/headlessService: headless
    spec:
      accessCredentials:
      - sshPublicKey:
          propagationMethod:
            noCloud: {}
          source:
            secret:
              secretName: <your_pubkey_secret>
      architecture: amd64
      domain:
        devices:
          autoattachPodInterface: false
          disks:
          - bootOrder: 2
            name: rootdisk
          - bootOrder: 1 # Delete this device if using PXE
            cdrom:
              bus: sata
            name: <my-dv>
          interfaces:
          - bootOrder: 3 #Change bootorder if using PXE
            bridge: {}
            macAddress: 02:a1:3b:00:00:56 #MAC Address used in agent-config.yaml
            model: virtio
            name: <private_nic-Name>
        resources: {}
      networks:
      - multus:
          networkName: default/br1-private #Private Network name, see available NAD 
        name: <private_nic-Name>
      subdomain: headless
      volumes:
      - dataVolume:
          name: my-sno-vm-volume-blank
        name: rootdisk
      - name: <my-dv>  # Delete this device if using PXE
        persistentVolumeClaim:
          claimName: <my-dv> #Volume name used in the virtcrl upload command
```


Installation will begin, you can monitor using the VM console from Openshift or with openshift-install

```shell
$ ./openshift-install agent --dir install_dir/ wait-for install-complete
```


#### 6. Access the cluster

```shell
$ export KUBECONFIG=install_dir/auth/kubeconfig
$ oc get nodes
$ oc get co
````

OR

```shell
$ oc login https://api.sno-cluster.test.mmayeras.local.ocpv:6443 -u kubeadmin -p $(cat install_dir/auth/kubeadmin-password)
```





