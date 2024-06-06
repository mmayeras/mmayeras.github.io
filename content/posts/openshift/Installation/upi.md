---
title: "User-provisioned installation"
date: 2023-03-24
tags:
- openshift
categories:
- openshift
# resources:
# - name: "featured-image"
#   src: "ocp.png"

---

### UPI

1. PXE Config 
```
dnf install -y tftp-server syslinux-tftpboot httpd haproxy
wget https://www.kernel.org/pub/linux/utils/boot/syslinux/syslinux-6.03.tar.gz
wget https://raw.githubusercontent.com/leoaaraujo/openshift_pxe_boot_menu/main/files/bg-ocp.png -O /var/lib/tftpboot/bios/bg-ocp.png
tar xf syslinux-6.03.tar.gz 
cp syslinux-6.03/bios/core/pxelinux.0 /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/com32/elflink/ldlinux/ldlinux.c32 /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/com32/lib/libcom32.c32 /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/com32/libutil/libutil.c32 /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/memdisk/memdisk /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/com32/modules/poweroff.c32 /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/com32/modules/pxechn.c32 /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/com32/modules/reboot.c32 /var/lib/tftpboot/bios/
cp syslinux-6.03/bios/com32/menu/vesamenu.c32 /var/lib/tftpboot/bios/
 
cp syslinux-6.03/efi64/efi/syslinux.efi /var/lib/tftpboot/efi64/
cp syslinux-6.03/efi64/com32/elflink/ldlinux/ldlinux.e64 /var/lib/tftpboot/efi64/
cp syslinux-6.03/efi64/com32/lib/libcom32.c32 /var/lib/tftpboot/efi64/
cp syslinux-6.03/efi64/com32/libutil/libutil.c32 /var/lib/tftpboot/efi64/
cp syslinux-6.03/bios/memdisk/memdisk /var/lib/tftpboot/efi64/
cp syslinux-6.03/efi64/com32/modules/poweroff.c32 /var/lib/tftpboot/efi64/
cp syslinux-6.03/efi64/com32/modules/pxechn.c32 /var/lib/tftpboot/efi64/
cp syslinux-6.03/efi64/com32/modules/reboot.c32 /var/lib/tftpboot/efi64/
cp syslinux-6.03/efi64/com32/menu/vesamenu.c32 /var/lib/tftpboot/efi64/


# wget https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live-kernel-x86_64  -O /var/www/html/ 
# wget https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live-initramfs.x86_64.img -O /var/www/html/
# wget https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live-rootfs.x86_64.img **-O /var/www/html/
systemctl enable tftp.service httpd --now
```

/var/lib/tftpboot/efi64/pxelinux.cfg/default
```
UI vesamenu.c32
MENU BACKGROUND        bg-ocp.png  
MENU COLOR sel         4  #ffffff std 
MENU COLOR title       1  #ffffff  
TIMEOUT 120
PROMPT 0
MENU TITLE OPENSHIFT 4.x INSTALL BARE METAL PXE MENU 
LABEL INSTALL BOOTSTRAP   
  KERNEL http://192.168.0.10:8080/rhcos-live-kernel-x86_64
  APPEND initrd=http://192.168.0.10:8080/rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.0.10:8080/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.0.10:8080/bootstrap.ign
LABEL INSTALL MASTER     
  KERNEL http://192.168.0.10:8080/rhcos-live-kernel-x86_64
  APPEND initrd=http://192.168.0.10:8080/rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.0.10:8080/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.0.10:8080/master.ign
LABEL INSTALL WORKER    
  KERNEL http://192.168.0.10:8080/rhcos-live-kernel-x86_64
  APPEND initrd=http://192.168.0.10:8080/rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.0.10:8080/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.0.10:8080/worker.ign
LABEL INSTALL EL
  KERNEL http://192.168.0.10:8080/el/vmlinuz
  APPEND initrd=http://192.168.0.10:8080/el/initrd.img inst.repo=http://192.168.0.10:8080/el/Packages/ inst.ks=http://192.168.0.10:8080/el/kickstart.cfg 

```

2. Create ignition config from install-config.yaml
```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 1
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16

platform:
  none: {}

fips: false

pullSecret: 'pull_secret'
sshKey: 'ssh-rsa'
```

`$ openshift-install create ignition-configs --dir ./cluster`

Copy ign files to the webserver.

haproxy.cfg
```
# Global settings
#---------------------------------------------------------------------
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          300s
    timeout server          300s
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 20000

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /

frontend ocp_k8s_api_fe
    bind :6443
    default_backend ocp_k8s_api_be
    mode tcp
    option tcplog

backend ocp_k8s_api_be
    balance source
    mode tcp
#    server      bootstrap 192.168.0.200:6443 check
    server      master1 192.168.0.201:6443 check
    #server      master2 192.168.0.202:6443 check
    #server      master3 192.168.0.203:6443 check

frontend ocp_machine_config_server_fe
    bind :22623
    default_backend ocp_machine_config_server_be
    mode tcp
    option tcplog

backend ocp_machine_config_server_be
    balance source
    mode tcp
#    server      bootstrap 192.168.0.200:22623 check
    server      master1 192.168.0.201:22623 check
    #server      master2 192.168.0.202:22623 check
    #server      master3 192.168.0.203:22623 check

frontend ocp_http_ingress_traffic_fe
    bind :80
    default_backend ocp_http_ingress_traffic_be
    mode tcp
    option tcplog

backend ocp_http_ingress_traffic_be
    balance source
    mode tcp
    #server      master1 192.168.0.201:80 check
    #server      master2 192.168.0.202:80 check
    #server      master3 192.168.0.203:80 check
    server      worker1 192.168.0.204:443 check
    server      worker2 192.168.0.205:443 check
    server      worker3 192.168.0.206:443 check

frontend ocp_https_ingress_traffic_fe
    bind *:443
    default_backend ocp_https_ingress_traffic_be
    mode tcp
    option tcplog

backend ocp_https_ingress_traffic_be
    balance source
    mode tcp
    #server      master1 192.168.0.201:443 check
    #server      master2 192.168.0.202:443 check
    #server      master3 192.168.0.203:443 check
    server      worker1 192.168.0.204:443 check
    server      worker2 192.168.0.205:443 check
    server      worker3 192.168.0.206:443 check
```

/etc/named/zones/db.192.168.0

```
$TTL    604800
@       IN      SOA     dns-server.example.com. (
                  6     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800     ; Negative Cache TTL
)

; name servers - NS records
    IN      NS     dns-server.example.com.

; name servers - PTR records
10    IN    PTR    dns-server.example.com.

; OpenShift Container Platform Cluster - PTR records
200    IN    PTR    bootstrap.example.com.
201    IN    PTR    master1.example.com.
202    IN    PTR    master2.example.com.
203    IN    PTR    master3.example.com.
204    IN    PTR    worker1.example.com.
205    IN    PTR    worker2.example.com.
206    IN    PTR    worker3.example.com.
10    IN    PTR    api.example.com.
10    IN    PTR    api-int.example.com.



Boot each machine and select correct entry in the pxe menu.
Bootstrap first, then masters and workers.
Once masters are up, boot strap can be shutdown and removed from the haproxy pool.