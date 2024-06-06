---
title: "Qemu cloud-init"
date: 2023-02-14
categories:
- linux
tags:
- qemu
- virt
summary: Create VM from template
toc:
  enable: true
  auto: true
---

### <i class="fab fa-linux"> qemu commands  </i>



`$ qemu-img create -b Fedora-Cloud-Base-37-1.7.x86_64.qcow2 -f qcow2 -F qcow2 f37-1.qcow2 10G`

`$ virt-install --name=fedora37 --ram=2048 --vcpus=1 --import --disk path=f37-1.qcow2,format=qcow2  --os-variant=fedora37 --network bridge=virbr0,model=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole --cloud-init user-data=user-data,meta-data=meta-data`

`#--network network=net-work-name,mac=02:01:00:00:00:66 --cdrom /var/lib/libvirt/images/some_iso.iso`

{{< admonition >}}
network name to adapt
{{< /admonition >}}


###  <i class="fas fa-code">  cloud-init </i>
##### meta-data
```yaml
instance-id: fedora37
local-hostname: fedora37
```
##### user-data
```yaml
#cloud-config
users:
  - name: foo
    ssh_authorized_keys:
      - ssh-rsa 
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash
```

