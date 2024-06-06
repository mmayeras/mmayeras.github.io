---
title: "NFS defaults"
date: 2022-12-13
categories:
- openshift
tags:
- openshift
---


## **Default versions**  
Kernel versions 4.18 and above default to nfs 4.2.
The client will try in order 4.2 then 4.1 then 4.0.  
https://access.redhat.com/articles/6907891  
https://access.redhat.com/articles/3626571


## **Default mount options**

`rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,local_lock=none,addr=<nfs_server>`


## **Customize mount options**  
https://access.redhat.com/solutions/6065961


```
apiVersion: v1
kind: PersistentVolume
spec:
[...]
  mountOptions:
    - nfsvers=4.1
[...]
```


You can make a RPC call to the NFS server to get supported versions :

```
$ rpcinfo -p 192.168.0.10 | grep nfs
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
```

And adapt mount options according to your nfs server.

```
# mount -vvv -t nfs <nfs_server>:/nfs_share /mountPoint
mount.nfs: timeout set for Wed Feb 24 13:01:29 2021
mount.nfs: trying text-based options 'vers=4.2,addr=<nfs_server>,clientaddr=<nfs_client>'
mount.nfs: mount(2): Protocol family not supported
mount.nfs: Protocol family not supported
``` 
