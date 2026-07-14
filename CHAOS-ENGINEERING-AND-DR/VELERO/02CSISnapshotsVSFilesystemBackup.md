# Velero Volume Backup - CSI Snapshots vs File System Backup

How Velero actually captures your Persistent Volume DATA, not just the Kubernetes object definitions, with visual diagrams.

## Table of Contents
- [Two Ways to Back Up Volume Data](#two-ways-to-back-up-volume-data)
- [CSI Snapshots](#csi-snapshots)
- [File System Backup (Restic/Kopia)](#file-system-backup-resticKopia)
- [Choosing Between Them](#choosing-between-them)
- [VolumeSnapshotLocation](#volumesnapshotlocation)
- [Opting Pods In/Out of File System Backup](#opting-pods-inout-of-file-system-backup)
- [Data Mover (Snapshot Data Movement)](#data-mover-snapshot-data-movement)

---

## Two Ways to Back Up Volume Data

**Visual:**
```
┌───────────────────────────────────────────────────┐
│                Persistent Volume Data                │
└───────────────────────┬───────────────────────────┘
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
┌──────────────────────┐   ┌──────────────────────┐
│  CSI Snapshots          │   │  File System Backup     │
│  (cloud-native disk       │   │  (Restic / Kopia,         │
│   snapshot, e.g.            │   │   file-level copy of        │
│   EBS/PD/Azure Disk)          │   │   actual bytes)                │
└──────────────────────┘   └──────────────────────┘
   Fast, storage-native          Portable across any
   Point-in-time, block-level     storage backend, works
   Requires CSI driver support     with any CSI or even
                                    hostPath volumes
```

---

## CSI Snapshots

**Uses the Kubernetes CSI (Container Storage Interface) snapshot API to trigger a native, storage-provider snapshot - fastest and most efficient method when supported.**

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: my-velero-backups
```

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0,velero/velero-plugin-for-csi:v0.7.0 \
  --features=EnableCSI \
  --bucket my-velero-backups \
  --snapshot-location-config region=us-east-1
```

**Visual:**
```
velero backup create db-backup --include-namespaces my-app
        │
        ▼
┌─────────────────────────────────────┐
│ Velero detects PVCs backed by a CSI     │
│ driver that supports snapshots            │
│         │                                   │
│         ▼                                   │
│ Creates a VolumeSnapshot object                │
│         │                                       │
│         ▼                                        │
│ Cloud provider creates an actual disk               │
│ snapshot (e.g. EBS snapshot) - happens at             │
│ the storage layer, near-instant, no data                │
│ copied through the cluster network                        │
└─────────────────────────────────────┘

Requires:
- A CSI driver installed that supports VolumeSnapshots
- A VolumeSnapshotClass configured for that driver
- --features=EnableCSI flag on velero install
```

---

## File System Backup (Restic/Kopia)

**Copies the actual file contents from inside the volume, byte by byte, through the node-agent DaemonSet - works regardless of storage backend, including hostPath and NFS.**

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket my-velero-backups \
  --use-node-agent \
  --uploader-type=kopia
```

**Visual:**
```
velero backup create db-backup --include-namespaces my-app
        │
        ▼
┌─────────────────────────────────────┐
│ node-agent pod on the SAME node as the  │
│ target pod mounts the volume's data        │
│         │                                    │
│         ▼                                     │
│ Reads every file, chunks + deduplicates          │
│ (kopia/restic), uploads to object storage           │
└─────────────────────────────────────┘

Slower than CSI snapshots (actual data transfer),
but works EVERYWHERE - no CSI snapshot support needed.
Deduplication means incremental backups are efficient
(only changed chunks are re-uploaded).

uploader-type options: restic (legacy) | kopia (newer, faster,
                         better dedup - recommended for new installs)
```

---

## Choosing Between Them

**Visual:**
```
                    CSI Snapshots          File System Backup
Speed                Fast (storage-native)  Slower (byte copy)
Portability           Cloud/storage-specific  Works everywhere
Incremental efficiency Depends on provider     Excellent (dedup)
Cross-cluster restore   Sometimes limited        Always works
Setup complexity          Requires CSI driver +     Simpler, just
                           VolumeSnapshotClass         --use-node-agent

Recommendation:
- Cloud-native cluster (EKS/GKE/AKS) with modern CSI drivers
  → prefer CSI snapshots for speed
- Migrating BETWEEN clusters/clouds, or storage without
  CSI snapshot support → File System Backup is the safe default
- Many production setups use BOTH: CSI for fast local DR,
  File System Backup for cross-region/cloud migration backups
```

---

## VolumeSnapshotLocation

**Configures WHERE CSI/native snapshots are stored - conceptually parallel to BackupStorageLocation, but for volume snapshots specifically.**

```yaml
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  config:
    region: us-east-1
```

```bash
velero snapshot-location get
```

**Visual:**
```
BackupStorageLocation   → where the JSON metadata/manifests go (object storage)
VolumeSnapshotLocation   → where the actual disk snapshots live (cloud provider's
                            native snapshot mechanism, e.g. EBS snapshots in your
                            AWS account)

Both are needed for a complete backup when using CSI/native snapshots.
```

---

## Opting Pods In/Out of File System Backup

**By default, File System Backup can be configured to back up ALL volumes or ONLY annotated ones - control this per-pod.**

```yaml
metadata:
  annotations:
    backup.velero.io/backup-volumes: "data-volume"
```

**Or opt a volume OUT:**

```yaml
metadata:
  annotations:
    backup.velero.io/backup-volumes-excludes: "cache-volume,tmp-volume"
```

**Visual:**
```
Pod: database-0
  volumes:
    - data-volume    ← IMPORTANT, must back up
    - cache-volume    ← ephemeral, safe to skip
    - tmp-volume       ← ephemeral, safe to skip

backup.velero.io/backup-volumes-excludes: "cache-volume,tmp-volume"
        │
        ▼
Only data-volume gets File System Backup treatment
→ faster backups, smaller storage footprint,
  skips volumes that don't need durability anyway
```

---

## Data Mover (Snapshot Data Movement)

**Newer Velero feature: moves CSI snapshot data INTO your object storage bucket (rather than leaving it only as a cloud-native snapshot) - combines speed of CSI with the portability of object storage.**

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: datamover-backup
  namespace: velero
spec:
  includedNamespaces:
    - my-app
  snapshotMoveData: true
```

**Visual:**
```
Without Data Mover:
CSI snapshot → stays in cloud provider's snapshot storage
              (tied to that specific cloud account/region)

With Data Mover (snapshotMoveData: true):
CSI snapshot → data copied INTO your Velero object storage
              bucket (portable, can restore to a totally
              different cloud/account/region)

Use case: true cross-cloud disaster recovery, where a CSI
snapshot alone wouldn't be restorable outside the original
cloud account.
```

---

## Visual Summary

```
Volume Backup Method Decision Tree:

Need fastest backup, staying within same cloud/account?
  → CSI Snapshots

Need to restore into a DIFFERENT cluster, cloud, or account?
  → File System Backup (Restic/Kopia)
    OR CSI Snapshots + Data Mover (snapshotMoveData: true)

Storage backend doesn't support CSI snapshots (NFS, hostPath)?
  → File System Backup is your only option

Want smaller/faster backups by skipping ephemeral volumes?
  → backup.velero.io/backup-volumes-excludes annotation
```

---

This guide covers how Velero backs up actual Persistent Volume data - CSI snapshots, File System Backup via Restic/Kopia, and the newer Data Mover feature for full cross-cloud portability, with visual representations of each mechanism.