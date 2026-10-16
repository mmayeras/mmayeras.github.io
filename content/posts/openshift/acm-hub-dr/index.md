---
title: "ACM hub disaster recovery: passive copy or active move to a new hub"
date: 2026-10-15
draft: false
categories:
  - Openshift
tags:
  - openshift
  - acm
  - backup
  - oadp
  - velero
  - disaster-recovery
---

High-level walkthrough of a guided lab that backs up a running Red Hat Advanced Cluster Management (ACM) 2.17 hub and restores it onto a brand-new Single Node OpenShift (SNO) hub while the primary stays up, in two scenarios: **passive** (copy the hub, the managed clusters stay where they are) and **active** (the managed clusters move to the new hub). The full lab (modules, manifests, scripts and an animated flow diagram) lives in the [acm-dr repository](https://github.com/mmayeras/acm-dr).

<!--more-->

*Acronyms: ACM = Advanced Cluster Management, SNO = Single Node OpenShift, OADP = OpenShift API for Data Protection.*

{{< admonition info "Lab repository: github.com/mmayeras/acm-dr" >}}
Every manifest, command and expected output referenced below lives in **[https://github.com/mmayeras/acm-dr](https://github.com/mmayeras/acm-dr)**:

- [`labs/`](https://github.com/mmayeras/acm-dr/tree/main/labs): ten guided modules, each ending with a checkpoint.
- [`manifests/`](https://github.com/mmayeras/acm-dr/tree/main/manifests): the DataProtectionApplication, BackupSchedule and all five Restore specifications, with every field commented.
- [`REFERENCE.md`](https://github.com/mmayeras/acm-dr/blob/main/REFERENCE.md#restore-spec-field-by-field): Restore fields explained one by one.
- [Animated flow diagram](https://mmayeras.fr/acm-dr/flow/): seven flows, step by step, including the passive standby where the spoke stays on HUB1.
{{< /admonition >}}

The lab follows the product procedure [Keeping the primary hub cluster active during a restore process](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/business_continuity/index#keep-hub-active-restore). ACM builds on the upstream [Open Cluster Management](https://github.com/open-cluster-management-io) project, and hub backup relies on the [cluster-backup-operator](https://github.com/stolostron/cluster-backup-operator) driving [Velero](https://github.com/vmware-tanzu/velero) through [OADP](https://github.com/openshift/oadp-operator).

## Topology

*Acronyms: ACM = Advanced Cluster Management, SNO = Single Node OpenShift, OADP = OpenShift API for Data Protection, CSV = ClusterServiceVersion (installed operator version), S3 = Amazon Simple Storage Service (object storage protocol).*

```text
                 S3 bucket (shared prefix, velero backups)
                  ^ write                    read |
                  |                               v
   +--------------+-----------+       +-----------+--------------+
   | HUB1 (existing, primary) |       | HUB2 (new SNO)           |
   | ACM 2.17.z               |       | ACM 2.17.z (same CSV)    |
   | OADP + BackupSchedule    |       | OADP, NO BackupSchedule  |
   +------------+-------------+       +-----------+--------------+
                |  klusterlet before              |  klusterlet after (active only)
                +-------------+    +--------------+
                              |    |
                         +----+----+----+
                         |    SPOKE     |
                         +--------------+
```

{{< admonition tip "Animated flow" >}}
The repository ships a FlowStory diagram with seven flows, ordered by scenario: backup on HUB1; **passive** (no cluster import, the spoke stays on HUB1), one-shot or continuous; **active** (cluster import, the spoke moves to HUB2), from a fresh restore or after a passive one, with the HUB1 preparation it requires; and, last, troubleshooting a spoke stuck in `Pending Import`. Open it at [mmayeras.fr/acm-dr/diagrams/acm-dr-flow.html](https://mmayeras.fr/acm-dr/diagrams/acm-dr-flow.html).
{{< /admonition >}}

## Prerequisites

*Acronyms: ACM = Advanced Cluster Management, SNO = Single Node OpenShift, S3 = Amazon Simple Storage Service (object storage protocol), API = Application Programming Interface.*

- An existing ACM 2.17 hub (HUB1) with at least one imported managed cluster.
- A fresh SNO cluster (HUB2) sized at least like HUB1, with no ACM installed.
- An S3-compatible object storage bucket reachable from both hubs.
- Active scenario only: network access open in both directions between HUB2 and the managed cluster (spoke), on the Kubernetes API server port `6443`:
  - **HUB2 → spoke API server**: during the restore, HUB2 logs in to the spoke with the backed-up token and points the spoke's agent to HUB2 (auto-import).
  - **Spoke → HUB2 API server**: the spoke's agent (klusterlet) then registers with HUB2 and keeps reporting to it.
- `oc`, `jq`, `envsubst` available in `$PATH`, with one kubeconfig context per cluster.

## 1. Enable backup on HUB1

*Acronyms: ACM = Advanced Cluster Management, OADP = OpenShift API for Data Protection, S3 = Amazon Simple Storage Service (object storage protocol), AWS = Amazon Web Services.*

Three objects on HUB1: the backup component, the storage location, the schedule. Full commands with checks: [module 2](https://github.com/mmayeras/acm-dr/blob/main/labs/02-enable-backup-hub1.md).

**Turn on the `cluster-backup` component** of the **MultiClusterHub**. It installs the ACM backup controller and the OADP operator in `open-cluster-management-backup`:

```bash
oc patch multiclusterhub multiclusterhub -n open-cluster-management --type json \
  -p '[{"op":"add","path":"/spec/overrides/components/-","value":{"name":"cluster-backup","enabled":true}}]'
```

If `cluster-backup` is already listed in `spec.overrides.components` with `enabled: false`, replace that entry instead of adding one (the module handles both cases).

**Store the bucket credentials** in the secret the storage location reads (key `cloud`, AWS credentials file format, which any S3-compatible storage accepts):

```bash
cat > credentials-velero <<EOF
[default]
aws_access_key_id=<access-key>
aws_secret_access_key=<secret-key>
EOF
oc create secret generic cloud-credentials -n open-cluster-management-backup \
  --from-file=cloud=credentials-velero
```

**Point a DataProtectionApplication at the bucket.** Object definitions only, no volume data, so no node agent and no image backup. Same file on both hubs ([`dpa.yaml.tpl`](https://github.com/mmayeras/acm-dr/blob/main/manifests/common/dpa.yaml.tpl)):

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-acm
  namespace: open-cluster-management-backup
spec:
  backupImages: false
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - aws
    nodeAgent:
      enable: false
      uploaderType: kopia
  backupLocations:
    - velero:
        provider: aws
        default: true
        objectStorage:
          bucket: <bucket>
          prefix: acm
        config:
          region: us-east-1
          s3ForcePathStyle: "true"
          s3Url: https://<s3-endpoint>
        credential:
          name: cloud-credentials
          key: cloud
```

The `aws` plugin talks to any S3-compatible endpoint (MinIO, OpenShift Data Foundation). For AWS itself, drop `s3Url` and `s3ForcePathStyle`. The storage location is ready when `oc get backupstoragelocation -n open-cluster-management-backup` shows `Available`.

**Create the BackupSchedule** ([`backupschedule.yaml.tpl`](https://github.com/mmayeras/acm-dr/blob/main/manifests/common/backupschedule.yaml.tpl)):

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: BackupSchedule
metadata:
  name: schedule-acm-msa
  namespace: open-cluster-management-backup
spec:
  veleroSchedule: "0 */1 * * *"
  veleroTtl: 120h
  useManagedServiceAccount: true
  paused: false
```

`useManagedServiceAccount: true` is the key field. For every imported cluster, the hub creates a **ManagedServiceAccount** whose token lands in the backup. Without it, imported clusters restored on HUB2 stay in `Pending Import` and need a manual re-import.

## 2. Install ACM on HUB2, pinned to the same CSV

*Acronyms: ACM = Advanced Cluster Management, OADP = OpenShift API for Data Protection, CSV = ClusterServiceVersion (installed operator version).*

HUB2 must run the exact same ACM version as HUB1. The lab reads the CSV from HUB1 and sets it as `startingCSV` on HUB2, then configures OADP against the same bucket and prefix. No **BackupSchedule** is created on HUB2: only one hub may write to the storage location, otherwise `BackupCollision`.

## 3. Choose what the restore brings back

Each **Restore** field selects one backup group (`latest`, `skip`, or a backup name), plus a cleanup policy. The managed-clusters field is the switch between the two scenarios:

| Field | Backup group | Role |
|---|---|---|
| `veleroCredentialsBackupName` | `acm-credentials-schedule` | hub data |
| `veleroResourcesBackupName` | `acm-resources-schedule` + `acm-resources-generic-schedule` | hub data, including any object labeled `cluster.open-cluster-management.io/backup` |
| `veleroManagedClustersBackupName` | `acm-managed-clusters-schedule` | `skip`: passive, the clusters stay. `latest`: active, the clusters move |
| `cleanupBeforeRestore` | - | `None` on an empty hub, `CleanupRestored` when the hub already holds restored data |

All fields are explained in [`REFERENCE.md`](https://github.com/mmayeras/acm-dr/blob/main/REFERENCE.md#restore-spec-field-by-field).

## 4. Passive scenario: copy the hub, the spoke stays

*Acronyms: RPO = Recovery Point Objective.*

With `veleroManagedClustersBackupName: skip` ([`restore-acm-passive.yaml`](https://github.com/mmayeras/acm-dr/blob/main/manifests/restore/restore-acm-passive.yaml), [`restore-acm-passive-sync.yaml`](https://github.com/mmayeras/acm-dr/blob/main/manifests/restore/restore-acm-passive-sync.yaml)), HUB2 receives policies, placements and credentials, but no cluster moves and HUB1 keeps running and backing up. Nothing to prepare on HUB1. Two flavours:

| | One-shot | Continuous |
|---|---|---|
| Extra fields | none | `syncRestoreWithNewBackups: true`, `restoreSyncInterval: 10m`, `CleanupRestored` |
| After the first restore | `Finished`, never reads newer backups | `Enabled`, restores each newer backup |
| Use for | point-in-time copy | warm standby |

A change on HUB1 reaches a continuous standby after the next HUB1 backup plus up to one sync interval: that delay is the RPO of a passive hub. Switching to the active scenario later means preparing HUB1 (step 5), then setting `veleroManagedClustersBackupName: latest` on HUB2.

## 5. Active scenario: prepare HUB1, which stays up

*Acronyms: RPO = Recovery Point Objective.*

- Pause the **BackupSchedule** on HUB1. The Velero schedules stay in place with `paused: true`, no new backup is written, and the last one defines the RPO.
- Apply a **Restore** on HUB1 itself. This tags HUB1 objects with `velero.io/backup-name`, which later enables a clean `CleanupRestored` failback.
- Annotate each moving **ManagedCluster** with `import.open-cluster-management.io/disable-auto-import` so HUB1 never pulls it back.

## 6. Active scenario: restore with the clusters

*Acronyms: MSA = ManagedServiceAccount.*

[`restore-acm.yaml`](https://github.com/mmayeras/acm-dr/blob/main/manifests/restore/restore-acm.yaml), applied on HUB2 (same object as the tagging restore of step 5):

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Restore
metadata:
  name: restore-acm
  namespace: open-cluster-management-backup
spec:
  cleanupBeforeRestore: None
  veleroManagedClustersBackupName: latest
  veleroCredentialsBackupName: latest
  veleroResourcesBackupName: latest
```

The restore controller finds the MSA token, creates an `auto-import-secret`, and the import controller uses it to rewrite the spoke `bootstrap-hub-kubeconfig`. The klusterlet then re-registers on HUB2. The token only grants get/update on that one secret, not cluster-admin.

Once HUB2 owns the clusters, it gets its own **BackupSchedule** and HUB1 releases the moved **ManagedCluster** objects. An optional module moves everything back to HUB1 with a `CleanupRestored` restore.

{{< admonition warning >}}
Check `spec.preserveOnDelete` on every Hive **ClusterDeployment** before deleting a **ManagedCluster** on HUB1. Otherwise, the real cluster gets deprovisioned.
{{< /admonition >}}

## Verify

*Acronyms: RPO = Recovery Point Objective.*

```bash
oc --context hub2 get managedcluster
oc --context spoke -n open-cluster-management-agent get secret hub-kubeconfig-secret \
  -o jsonpath='{.data.kubeconfig}' | base64 -d | grep server:
```

Expected output, active scenario:

```text
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true           https://api.hub2...    True     True        2h
spoke1          true           https://api.spoke1...  True     True        12m
    server: https://api.<hub2-domain>:6443
```

The spoke shows `Unknown` on HUB1. Anything created on HUB1 after the last backup is missing on HUB2: the RPO equals the time since the last backup.

Expected output, passive scenario: only `local-cluster` on HUB2, and the spoke's `server:` still points to HUB1.
