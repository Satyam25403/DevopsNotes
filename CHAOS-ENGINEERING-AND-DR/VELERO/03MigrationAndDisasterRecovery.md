# Velero Migration & Disaster Recovery - Visual Guide

Using Velero for its two biggest real-world use cases: moving workloads between clusters, and recovering from disaster.

## Table of Contents
- [Cluster Migration Pattern](#cluster-migration-pattern)
- [Cross-Cloud / Cross-Region Migration](#cross-cloud--cross-region-migration)
- [StorageClass Mapping During Migration](#storageclass-mapping-during-migration)
- [Full Disaster Recovery Scenario](#full-disaster-recovery-scenario)
- [Namespace-Level Recovery Scenario](#namespace-level-recovery-scenario)
- [Pre-Upgrade Safety Net Pattern](#pre-upgrade-safety-net-pattern)
- [Testing Your DR Plan (the part everyone skips)](#testing-your-dr-plan-the-part-everyone-skips)

---

## Cluster Migration Pattern

**Moving workloads from an old cluster to a new one - version upgrade, cloud provider switch, or region change.**

**Visual:**
```
Old Cluster (source)                  New Cluster (destination)
┌─────────────────────┐              ┌─────────────────────┐
│ Namespace: my-app       │              │ Namespace: my-app       │
│  frontend, backend,       │              │  (empty, freshly           │
│  database, PVCs               │  ──────→   │   created cluster)          │
└─────────────────────┘   backup/    └─────────────────────┘
                            restore

Steps:
1. velero backup create migration-backup --include-namespaces my-app
   (on OLD cluster's Velero instance)
2. Point Velero on the NEW cluster at the SAME object storage bucket
   (same BackupStorageLocation config)
3. velero backup get   (on NEW cluster - it can see the backup!)
4. velero restore create --from-backup migration-backup
   (on NEW cluster)
5. Verify workloads running, cut over DNS/traffic
```

**Visual - shared bucket is the key:**
```
┌────────────────────────────────────────┐
│      Shared Object Storage Bucket          │
│         s3://my-velero-backups                │
└───────────┬────────────────┬───────────┘
            │                │
   backup   │                │  restore
            ▼                ▼
    ┌───────────────┐  ┌───────────────┐
    │  Old Cluster     │  │  New Cluster     │
    │  Velero            │  │  Velero            │
    └───────────────┘  └───────────────┘

Both clusters' Velero installs point at the SAME bucket -
this is what makes cross-cluster restore possible at all.
```

---

## Cross-Cloud / Cross-Region Migration

**Migrating between different cloud providers (e.g. AWS → GCP) requires File System Backup or Data Mover, since native CSI snapshots don't transfer between clouds.**

**Visual:**
```
AWS Cluster (EBS-backed PVCs)         GCP Cluster (PD-backed PVCs)
┌─────────────────────┐              ┌─────────────────────┐
│ velero install            │              │ velero install            │
│  --provider aws              │              │  --provider gcp              │
│  --use-node-agent               │              │  --use-node-agent               │
│  --uploader-type=kopia             │              │  --uploader-type=kopia             │
└─────────────────────┘              └─────────────────────┘
            │                                       ▲
            │ backup (File System Backup,             │ restore (recreates
            │  actual file bytes, not an                │  PVCs as native GCP
            │  EBS snapshot)                                │  Persistent Disks)
            ▼                                       │
┌────────────────────────────────────────┐
│  Shared Object Storage (works cross-cloud, │
│  e.g. an S3 bucket both clusters can reach)   │
└────────────────────────────────────────┘

Key point: CSI snapshots are tied to the ORIGINAL cloud's
snapshot mechanism (EBS snapshots only work in AWS) - for
true cross-cloud migration, use File System Backup or
Data Mover (snapshotMoveData: true) instead.
```

---

## StorageClass Mapping During Migration

**The destination cluster likely has different StorageClass names - map them during restore.**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: change-storage-class-config
  namespace: velero
  labels:
    velero.io/plugin-config: ""
    velero.io/change-storage-class: RestoreItemAction
data:
  gp2: premium-rwo
```

**Visual:**
```
Old Cluster (AWS)              New Cluster (GCP)
StorageClass: gp2       →      StorageClass: premium-rwo

Without this mapping: restore fails because "gp2" StorageClass
doesn't exist on the destination cluster.

With the ConfigMap: Velero automatically rewrites PVC specs
during restore to use the equivalent StorageClass.
```

---

## Full Disaster Recovery Scenario

**Entire cluster lost (region outage, accidental deletion, catastrophic failure) - recover everything from scratch.**

**Visual:**
```
┌────────────────────────────────────────┐
│ INCIDENT: entire cluster is gone            │
└────────────────────────────────────────┘
                    │
                    ▼
Step 1: Provision a new cluster
        (Terraform/eksctl/whatever created the original)

Step 2: Install Velero, pointing at the SAME backup bucket
        velero install --provider aws --bucket my-velero-backups ...

Step 3: Confirm backups are visible
        velero backup get
        NAME                    STATUS      CREATED
        nightly-backup-071426   Completed   2026-07-14 02:00

Step 4: Restore the most recent good backup
        velero restore create --from-backup nightly-backup-071426

Step 5: Verify application health, DNS/ingress cutover,
        confirm data integrity (especially DB consistency
        if backup hooks were used)

Recovery Time = time to provision cluster + time to restore
                (usually minutes to low hours, NOT days)
```

---

## Namespace-Level Recovery Scenario

**More common than full cluster loss - someone `kubectl delete namespace`'d the wrong thing, or a bad Helm upgrade corrupted resources.**

```bash
velero restore create --from-backup nightly-backup-071426 \
  --include-namespaces my-app
```

**Visual:**
```
┌────────────────────────────────────────┐
│ INCIDENT: namespace my-app accidentally      │
│ deleted (kubectl delete ns my-app)              │
└────────────────────────────────────────┘
                    │
                    ▼
velero restore create --from-backup nightly-backup-071426 \
  --include-namespaces my-app
                    │
                    ▼
Namespace, all Deployments/Services/ConfigMaps/PVCs
recreated from the most recent backup that included it

Recovery Time = restore duration only (usually minutes)
This is the MOST common real-world Velero recovery scenario -
far more common than losing an entire cluster.
```

---

## Pre-Upgrade Safety Net Pattern

**Take an on-demand backup immediately before any risky change - cluster upgrade, Helm chart major version bump, schema migration.**

**Visual:**
```
┌──────────────────────────────────────────────────┐
│ Pipeline: pre-upgrade-safety-net                       │
├──────────────────────────────────────────────────┤
│ 1. velero backup create pre-upgrade-$(date +%s) \      │
│      --include-namespaces my-app                         │
│ 2. velero backup describe <name> --details               │
│    (confirm Phase: Completed, Errors: 0 before proceeding) │
│ 3. Proceed with the risky change                            │
│    (helm upgrade / kubectl apply / cluster version bump)      │
│ 4. If something breaks:                                          │
│    velero restore create --from-backup pre-upgrade-<ts>            │
└──────────────────────────────────────────────────┘

This turns "we hope the upgrade works" into "we have an
exact rollback point no matter what happens."
```

---

## Testing Your DR Plan (the part everyone skips)

**A backup you've never restored from is a backup you don't actually have.**

**Visual:**
```
DR Test Runbook (run quarterly, minimum):
┌──────────────────────────────────────────────────┐
│ 1. Spin up a throwaway test cluster                    │
│ 2. Point Velero at production's backup bucket             │
│    (read-only credentials if possible)                       │
│ 3. Restore the latest production backup into it              │
│ 4. Verify:                                                       │
│    - All expected namespaces/resources present                    │
│    - PV data restored and readable                                  │
│    - Application actually starts and passes health checks             │
│ 5. Tear down the test cluster                                          │
│ 6. Document restore time (your actual measured RTO)                       │
└──────────────────────────────────────────────────┘

Untested backups are a false sense of security - the
restore process itself (StorageClass mapping, hooks,
CSI driver availability) can silently break over time
as your cluster configuration drifts.
```

---

## Visual Summary

```
1. Cluster migration        → same bucket, backup on old, restore on new
2. Cross-cloud migration      → File System Backup or Data Mover required
3. StorageClass mapping         → ConfigMap rewrites PVC specs during restore
4. Full DR                        → new cluster + restore from most recent backup
5. Namespace recovery                → most common scenario, fast targeted restore
6. Pre-upgrade safety net               → on-demand backup before risky changes
7. DR testing                             → restore into a throwaway cluster regularly
```

---

This guide covers Velero's disaster recovery and migration patterns - cross-cluster restore, cross-cloud considerations, and the DR testing discipline that makes backups actually trustworthy, with visual representations of each scenario.