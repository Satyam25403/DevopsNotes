# Velero Basics - What It Is, Architecture, Install

Essential Velero concepts and setup commands for backing up and restoring Kubernetes clusters, with visual diagrams.

## Table of Contents
- [What is Velero](#what-is-velero)
- [What Velero Backs Up](#what-velero-backs-up)
- [Architecture](#architecture)
- [Install the CLI](#install-the-cli)
- [Install the Server (into the cluster)](#install-the-server-into-the-cluster)
- [Verify Installation](#verify-installation)
- [Your First Backup](#your-first-backup)

---

## What is Velero

**Velero is an open-source tool for backing up, restoring, and migrating Kubernetes cluster resources and persistent volumes. It's the standard answer to "how do you do disaster recovery for Kubernetes?"**

**Visual:**
```
Without Velero:
┌────────────────────────────────────┐
│ Cluster deleted / namespace         │
│ accidentally wiped / region outage   │
└────────────────────────────────────┘
             │
             ▼
     Rebuild everything from memory,
     Git repos, and hope nothing was missed
     ┌─────────────────────┐
     │  Hours/days of        │
     │  manual recovery       │
     └─────────────────────┘

With Velero:
┌────────────────────────────────────┐
│ Cluster deleted / namespace          │
│ accidentally wiped / region outage    │
└────────────────────────────────────┘
             │
             ▼
     velero restore create --from-backup <name>
     ┌─────────────────────┐
     │  Minutes, automated,   │
     │  consistent recovery    │
     └─────────────────────┘
```

**What it's used for:**
- Disaster recovery (cluster or namespace loss)
- Cluster migration (move workloads to a new cluster/cloud/region)
- Scheduled, ongoing backups as an insurance policy
- Cloning a namespace/environment (e.g. prod → staging for testing)
- Pre-upgrade safety net before risky cluster changes

---

## What Velero Backs Up

**Visual:**
```
┌───────────────────────────────────────────┐
│              A Velero Backup                │
├───────────────────────────────────────────┤
│ 1. Kubernetes API objects (as JSON)           │
│    Deployments, Services, ConfigMaps,          │
│    Secrets, PVCs, CRDs, RBAC, etc.              │
│                                                │
│ 2. Persistent Volume data (optional)            │
│    via CSI snapshots (cloud disk snapshots)       │
│    or File System Backup (Restic/Kopia,             │
│    copies actual file bytes)                          │
└───────────────────────────────────────────┘

Both pieces are needed for a FULL recovery:
API objects alone = empty PVCs on restore (no data)
Volume data alone  = no idea how to reattach it to pods
```

---

## Architecture

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                  Namespace: velero                          │
│                                                              │
│  ┌────────────────┐        ┌──────────────────────────┐    │
│  │ velero (Deploy)  │        │ node-agent (DaemonSet)     │    │
│  │ - watches Backup/ │        │ - runs on every node        │    │
│  │   Restore CRDs      │        │ - performs File System       │    │
│  │ - talks to cloud     │        │   Backup (Restic/Kopia)       │    │
│  │   provider plugin      │        │   for PV data                  │    │
│  └────────┬───────┘        └──────────────────────────┘    │
│           │                                                    │
└───────────┼──────────────────────────────────────────────────┘
            │ stores backups in
            ▼
┌─────────────────────────────────────────────────────────┐
│         Object Storage (S3, GCS, Azure Blob, MinIO)          │
│  ┌─────────────────────────────────────────┐                │
│  │ backups/                                    │               │
│  │   nightly-backup-2026-07-14/                  │              │
│  │     resources.json                              │             │
│  │     volumesnapshots.json                          │           │
│  └─────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

**Key components:**
| Component | Role |
|---|---|
| `velero` (server Deployment) | Core controller - watches Backup/Restore/Schedule CRDs, orchestrates the process |
| `node-agent` (DaemonSet) | Performs File System Backup (Restic/Kopia) of PV data, one pod per node |
| `velero` CLI | Local tool to trigger backups/restores, talks to the server via CRDs |
| Object Storage plugin | Cloud-provider-specific plugin (AWS/GCP/Azure/MinIO) for storing backup data |
| CSI plugin (optional) | Enables native cloud disk snapshots instead of/alongside file backup |

---

## Install the CLI

```bash
# macOS
brew install velero

# Linux (manual)
wget https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xvf velero-v1.14.0-linux-amd64.tar.gz
sudo mv velero-v1.14.0-linux-amd64/velero /usr/local/bin/
```

```bash
velero version --client-only
```

**Visual:**
```
Your Machine
┌─────────────────┐
│ /usr/local/bin/   │
│   └── velero       │  ← CLI binary
└─────────────────┘

CLI talks to the cluster via your kubeconfig, same as kubectl
```

---

## Install the Server (into the cluster)

**Requires an object storage bucket already created (S3/GCS/Azure Blob/MinIO) before running install.**

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero \
  --use-node-agent
```

**Visual:**
```
Step 1: Create object storage bucket (outside Velero)
  aws s3 mb s3://my-velero-backups

Step 2: Create credentials file
  credentials-velero:
    [default]
    aws_access_key_id=...
    aws_secret_access_key=...

Step 3: velero install
  ┌──────────────────────────────────┐
  │ namespace: velero created            │
  │  velero Deployment                    │
  │  node-agent DaemonSet (--use-node-agent)│
  │  BackupStorageLocation "default"          │
  │  VolumeSnapshotLocation "default"           │
  └──────────────────────────────────┘

--use-node-agent → only needed if you want File System Backup
                    (Restic/Kopia); skip if using CSI snapshots only
```

**Other providers follow the same pattern:**
```bash
# GCP
velero install --provider gcp --plugins velero/velero-plugin-for-gcp:v1.10.0 \
  --bucket my-velero-backups --secret-file ./credentials-velero

# Azure
velero install --provider azure --plugins velero/velero-plugin-for-microsoft-azure:v1.10.0 \
  --bucket my-velero-backups --secret-file ./credentials-velero \
  --backup-location-config resourceGroup=...,storageAccount=...

# MinIO / on-prem S3-compatible
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket my-velero-backups --secret-file ./credentials-velero \
  --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://minio.example.com:9000
```

---

## Verify Installation

```bash
kubectl get pods -n velero
velero backup-location get
```

**Output Example:**
```
NAME                     READY   STATUS
velero-7f8...            1/1     Running
node-agent-4x9j2         1/1     Running
node-agent-7k2m1         1/1     Running

NAME      PROVIDER   BUCKET/PREFIX         PHASE       LAST VALIDATED
default   aws        my-velero-backups     Available   5s ago
```

**Visual:**
```
PHASE: Available  → Velero can reach the object storage bucket
PHASE: Unavailable → check credentials/bucket permissions/network
```

---

## Your First Backup

```bash
velero backup create my-first-backup --include-namespaces my-app
velero backup describe my-first-backup
velero backup logs my-first-backup
```

**Visual:**
```
kubectl get all -n my-app     (before)
frontend, backend, database all running

velero backup create my-first-backup --include-namespaces my-app
        │
        ▼
┌─────────────────────────────────┐
│ Backup: my-first-backup            │
│  Phase: Completed                    │
│  Items backed up: 24                   │
│  Errors: 0                              │
└─────────────────────────────────┘
        │
        ▼
Stored in s3://my-velero-backups/backups/my-first-backup/
```

---

## Visual Summary

```
1. Install CLI
   brew install velero  /  download binary

2. Create object storage bucket (outside Velero)
   aws s3 mb s3://my-velero-backups

3. Install server into cluster
   velero install --provider aws --bucket ... --use-node-agent

4. Verify
   kubectl get pods -n velero
   velero backup-location get

5. Run first backup
   velero backup create my-first-backup --include-namespaces my-app

6. Verify it worked
   velero backup describe my-first-backup
```

**Core Idea:**
```
┌────────────────┐   backup    ┌────────────────┐   restore   ┌────────────────┐
│  Live Cluster   │  ────────→  │  Object Storage │  ─────────→  │  Same or New    │
│  (source truth) │             │  (safe copy)    │              │  Cluster        │
└────────────────┘             └────────────────┘             └────────────────┘
```

---

This guide covers Velero basics: what it backs up, its architecture, and installing it with your first backup, with visual representations of each step.