---
title: "Ceph installation"
date: 2023-02-15
tags:
- openshift
- storage
categories:
- openshift

---

### Requirements
    Red Hat Enterprise Linux 8.4 EUS or later.
    Ansible 2.9 or later.
    A valid Red Hat subscription with the appropriate entitlements.
    Root-level access to all nodes.
    An active Red Hat Network (RHN) or service account to access the Red Hat Registry.

### Create 3 RHEL 8 virtual machines
   1. ceph1
   2. ceph2
   3. ceph3
    
{{< admonition >}}
_**Installing Ceph on Virtual Machines is not recommendend for production use**_

{{< /admonition >}}

### Register servers to RHN
### Find and attach Red Hat Ceph Storage pool  
   ```
   $ subscription-manager list --available --matches 'Red Hat Ceph Storage'    
   $ subscription-manager attach --pool=POOL_ID
   ```
### Enable server & extra repos
```
$subscription-manager repos --disable=*
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
subscription-manager repos --enable=rhceph-5-tools-for-rhel-8-x86_64-rpms
subscription-manager repos --enable=ansible-2.9-for-rhel-8-x86_64-rpms
```
### Update system  
`$ dnf update -y`

### Install cephadm-ansible & cephadm

### Create inventory file
/usr/share/cephadm-ansible/inventory
```yaml
ceph1
ceph2
ceph3

[admin]
ceph1
```

### Run the preflight playbook
   
   `$ ansible-playbook -i INVENTORY_FILE cephadm-preflight.yml --extra-vars "ceph_origin=rhcs"`

### Bootstrap cluster

```
$ NETWORK_CIDR=10.10.0.0/24
$ IP_ADDRESS=10.10.0.x/24
$ USER_NAME=rhn_username
$ read PASSWORD
mypassword
$ cephadm bootstrap --cluster-network $NETWORK_CIDR --mon-ip $IP_ADDRESS --registry-url registry.redhat.io --registry-username $USER_NAME --registry-password $PASSWORD --yes-i-know --allow-fqdn-hostname

Ceph Dashboard is now available at:

	     URL: https://ceph1:8443/
	    User: admin
	Password: whateverpassword

```

### Add hosts using the WebUI

### Optional: fix ssh key exchange
```
$ ceph cephadm get-pub-key > ~/ceph.pub 
$ ssh-copy-id -f -i ~/ceph.pub root@ceph2
$ ssh-copy-id -f -i ~/ceph.pub root@ceph3
```

### Add disks to the VMs and create OSD
### Add cephfs, rgw, rbd pools using the web UI

Go to [ODF installation](../odfinstall)


