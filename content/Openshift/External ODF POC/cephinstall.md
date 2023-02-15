+++
title = "Ceph installation"
date = 2023-02-15T09:39:20+01:00
weight = 5
chapter = false
hidden = false
+++

0. Requirements
    Red Hat Enterprise Linux 8.4 EUS or later.
    Ansible 2.9 or later.
    A valid Red Hat subscription with the appropriate entitlements.
    Root-level access to all nodes.
    An active Red Hat Network (RHN) or service account to access the Red Hat Registry.

1. Create 3 RHEL 8 virtual machines
   1. ceph1
   2. ceph2
   3. ceph3
    
---
_**Installing Ceph on Virtual Machines is not recommendend for production use**_

---
2. Register servers to RHN
3. Find and attach Red Hat Ceph Storage pool  
   ```
   $ subscription-manager list --available --matches 'Red Hat Ceph Storage'    
   $ subscription-manager attach --pool=POOL_ID
   ```
4. Enable server & extra repos
```
$subscription-manager repos --disable=*
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
subscription-manager repos --enable=rhceph-5-tools-for-rhel-8-x86_64-rpms
subscription-manager repos --enable=ansible-2.9-for-rhel-8-x86_64-rpms
```
5. Update system  
`$ dnf update -y`

6. Install cephadm-ansible & cephadm

7. Create inventory file
/usr/share/cephadm-ansible/inventory
```yaml
ceph1
ceph2
ceph3

[admin]
ceph1
```

8. Run the preflight playbook
   
   `$ ansible-playbook -i INVENTORY_FILE cephadm-preflight.yml --extra-vars "ceph_origin=rhcs"`

9. Bootstrap cluster

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

10. Add hosts using the WebUI

11. Optional: fix ssh key exchange
```
$ ceph cephadm get-pub-key > ~/ceph.pub 
$ ssh-copy-id -f -i ~/ceph.pub root@ceph2
$ ssh-copy-id -f -i ~/ceph.pub root@ceph3
```

12. Add disks to the VMs and create OSD
13. Add cephfs, rgw, rbd pools using the web UI


