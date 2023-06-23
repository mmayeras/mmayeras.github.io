---
title: "OCP etcd backup"
date: 2023-06-23
categories:
- openshift
tags:
- openshift
---



## How to schedule etcd backups using cronjob

What you will need :
- Namespace
- Service Account
- Cluster Role
- Cluster Role Binding
- Extend SA priviliges
- Cronjob


### Namespace

```yaml
---
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  annotations:
    openshift.io/description: Openshift Backup Automation Tool
    openshift.io/display-name: Backup ETCD Automation  
  name: ocp-etcd-backup
  finalizers:
  - kubernetes
```

### Service Account

```yaml
---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: openshift-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-backup
```

### Cluster role
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-etcd-backup
rules:
- apiGroups: [""]
  resources:
     - "namespaces"
  verbs: ["get", "list", "create"]

- apiGroups: [""]
  resources:
     - "nodes"
  verbs: ["get", "list"]
- apiGroups: [""]
  resources:
     - "pods"
     - "pods/log"
  verbs: ["get", "list", "create", "delete", "watch"]
```

### Cluster Role Binding
```yaml
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openshift-backup
  labels:
    app: openshift-backup
subjects:
  - kind: ServiceAccount
    name: openshift-backup
    namespace: ocp-etcd-backup
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-etcd-backup
```

### Cronjob
```yaml
---
kind: CronJob
apiVersion: batch/v1
metadata:
  name: openshift-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-backup
spec:
  schedule: "56 23 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    metadata:
      labels:
        app: openshift-backup
    spec:
      backoffLimit: 0
      template:
        metadata:
          labels:
            app: openshift-backup
        spec:
          containers:
            - name: backup
              image: "registry.redhat.io/openshift4/ose-cli"
              command:
                - "/bin/bash"
                - "-c"
                - oc get no -l node-role.kubernetes.io/master --no-headers -o name | xargs -I {} --  oc debug {} -- bash -c 'chroot /host sudo -E /usr/local/bin/cluster-backup.sh /home/core/backup/ && chroot /host sudo -E find /home/core/backup/ -type f -mmin +"1" -delete'
          restartPolicy: "Never"
          terminationGracePeriodSeconds: 30
          activeDeadlineSeconds: 500
          dnsPolicy: "ClusterFirst"
          serviceAccountName: "openshift-backup"
          serviceAccount: "openshift-backup"
```

{{< admonition info "Info" >}}
If you wish to backup to a NFS share, change the command into the backup container to :
```yaml
oc get no -l node-role.kubernetes.io/master --no-headers -o name | xargs -I {} --  oc debug {} -- bash -c 'chroot /host rm -rf /home/core/backup && chroot /host  mkdir /home/core/backup && chroot /host  sudo -E  mount -t nfs <nfs-server-IP>:<shared-path>    /home/core/backup && chroot /host sudo -E /usr/local/bin/cluster-backup.sh /home/core/backup && chroot /host sudo -E find /home/core/backup/ -type f -mmin +"1" -delete'
```
{{< /admonition >}}

### Test job execution

`oc create job backup --from=cronjob/openshift-backup`