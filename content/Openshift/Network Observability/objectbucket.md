+++
title = "Object Bucket Claim"
date = 2023-02-09T12:04:20+01:00
weight = 2
chapter = false
+++


#### In my LAB environment, I am using ODF Internal to provide object buckets.

#### First, let's create an object bucket claim.

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata: 
  name: netobserv-bucket 
spec:
  storageClassName: ocs-storagecluster-ceph-rgw
  generateBucketName: netobserv-bucket 
```

#### Get the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from the secret generated :

```yaml
#$ oc get secret netobserv-bucket -o yaml
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: NzI0MEtCNU5SVThVVkRGUFVCRVY=
  AWS_SECRET_ACCESS_KEY: NzI0MEtCNU5SVThVVkRGUFVCRVY=
kind: Secret
...
```
#### Get the generated bucket name and S3 endpoint from the Object Bucket :
```yaml
#$ oc get objectbucket netobserv-bucket -o yaml
...
spec:
    additionalState:
      cephUser: ceph-user-foQpCUE6
    claimRef: {}
    endpoint:
      additionalConfig: {}
      bucketHost: rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc
      bucketName: netobserv-bucket-0f653cc9-a18c-46a0-9316-d200302a7922
      bucketPort: 443
      region: ""
      subRegion: ""

...
```

#### Create a new secret using these values :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: network-observability-s3-secret
  namespace: netobserv
stringData:
  access_key_id: NzI0MEtCNU5SVThVVkRGUFVCRVY=
  access_key_secret: NzI0MEtCNU5SVThVVkRGUFVCRVY=
  endpoint: https://rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc
  bucketNames: netobserv-bucket-0f653cc9-a18c-46a0-9316-d200302a7922
  region: ''
```