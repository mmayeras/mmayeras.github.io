+++
title = "ODF installation"
date = 2023-02-15T09:39:20+01:00
weight = 6
chapter = false
hidden = false
+++


1. Install Openshift Data Foundation from Operator Hub

2. Create a StorageSystem using "Connect an external storage platform" of Red Hat Ceph Storage type
3. Download ceph-external-cluster-details-exporter.py script and run it on your ceph admin node

```
$ python3 ceph-external-cluster-details-exporter.py --rbd-data-pool-name testrbd --cephfs-data-pool-name cephfs.testfs.data --rgw-endpoint 10.0.0.n:80 --cephfs-filesystem-name testfs
```

Sample output :
```
[{"name": "rook-ceph-mon-endpoints", "kind": "ConfigMap", "data": {"data": "ceph1=10.0.0.n:6789", "maxMonId": "0", "mapping": "{}"}}, {"name": "rook-ceph-mon", "kind": "Secret", "data": {"admin-secret": "admin-secret", "fsid": "5dabcb8e-ad19-11ed-a179-005056af8aeb", "mon-secret": "mon-secret"}}, {"name": "rook-ceph-operator-creds", "kind": "Secret", "data": {"userID": "client.healthchecker", "userKey": "********************"}}, {"name": "rook-csi-rbd-node", "kind": "Secret", "data": {"userID": "csi-rbd-node", "userKey": "********"}}, {"name": "ceph-rbd", "kind": "StorageClass", "data": {"pool": "testrbd"}}, {"name": "monitoring-endpoint", "kind": "CephCluster", "data": {"MonitoringEndpoint": "10.0.0.n", "MonitoringPort": "9283"}}, {"name": "rook-ceph-dashboard-link", "kind": "Secret", "data": {"userID": "ceph-dashboard-link", "userKey": "https://10.0.0.n:8443/"}}, {"name": "rook-csi-rbd-provisioner", "kind": "Secret", "data": {"userID": "csi-rbd-provisioner", "userKey": "************"}}, {"name": "rook-csi-cephfs-provisioner", "kind": "Secret", "data": {"adminID": "csi-cephfs-provisioner", "adminKey": "***********"}}, {"name": "rook-csi-cephfs-node", "kind": "Secret", "data": {"adminID": "csi-cephfs-node", "adminKey": "*************"}}, {"name": "cephfs", "kind": "StorageClass", "data": {"fsName": "testfs", "pool": "cephfs.testfs.data"}}, {"name": "ceph-rgw", "kind": "StorageClass", "data": {"endpoint": "10.0.0.n:80", "poolPrefix": "default"}}, {"name": "rgw-admin-ops-user", "kind": "Secret", "data": {"accessKey": "************************", "secretKey": "**********************"}}]
```

4. Save the json file and import it in the StorageSystem wizard
   

**rbd pool must be replicated because Erasure-Coded RBD pool(s) are not supported in ODF.**

If you accidently created an erasure-coded pool, you can delete it and re run the ODF import script

```
$ ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
$ ceph osd pool rm test-pool test-pool --yes-i-really-really-mean-it
```

