# Velero Backup & Restore - Visual Guide

Day-to-day commands for creating, scheduling, and restoring backups - the core Velero workflow used in real production.

## Table of Contents
- [Creating a Backup](#creating-a-backup)
- [Include/Exclude Filters](#includeexclude-filters)
- [Scheduled Backups](#scheduled-backups)
- [Backup Retention (TTL)](#backup-retention-ttl)
- [Restoring a Backup](#restoring-a-backup)
- [Restore Namespace Mapping](#restore-namespace-mapping)
- [Backup Hooks](#backup-hooks)
- [Restore Hooks](#restore-hooks)

---

## Creating a Backup

```bash
velero backup create backend-backup --include-namespaces my-app
```

**Visual:**
```
velero backup create <name> [flags]
         │
         ▼
┌─────────────────────────────────┐
│ 1. Query K8s API for matching     │
│    resources                        │
│ 2. Serialize them to JSON            │
│ 3. Trigger CSI snapshot or File       │
│    System Backup for PVs               │
│ 4. Upload everything to object          │
│    storage bucket                        │
│ 5. Mark Backup CRD phase: Completed        │
└─────────────────────────────────┘
```

### Check status

```bash
velero backup get
velero backup describe backend-backup --details
```

**Output Example:**
```
NAME              STATUS      CREATED                 EXPIRES
backend-backup    Completed   2026-07-14 10:00:00     2026-08-13
```

---

## Include/Exclude Filters

**Scope exactly what a backup captures - critical for large clusters where a full-cluster backup is wasteful or too slow.**

```bash
velero backup create db-only-backup \
  --include-namespaces my-app \
  --include-resources deployments,services,persistentvolumeclaims \
  --exclude-resources events \
  --selector app=database
```

**Visual:**
```
--include-namespaces my-app        → only this namespace
--exclude-namespaces kube-system     → skip system namespaces
--include-resources deploy,svc,pvc     → only these resource types
--exclude-resources events               → skip noisy/useless resources
--selector app=database                    → only resources with this label

Full-cluster backup (no filters):
┌─────────────────────────────────┐
│ Every namespace, every resource     │  ← slow, large, often unnecessary
└─────────────────────────────────┘

Scoped backup (with filters):
┌─────────────────────────────────┐
│ Just what you actually need          │  ← fast, small, targeted
└─────────────────────────────────┘
```

---

## Scheduled Backups

**The Schedule CRD automates recurring backups - the standard production pattern (nightly, hourly, etc.).**

```bash
velero schedule create nightly-backup \
  --schedule="0 2 * * *" \
  --include-namespaces my-app \
  --ttl 720h0m0s
```

**Or via YAML:**

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: nightly-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - my-app
    ttl: 720h0m0s
```

**Visual:**
```
Every night at 02:00
┌─────────────────────────────────────────┐
│ Schedule: nightly-backup                    │
│ → creates a new Backup object automatically   │
│ → named nightly-backup-20260714020000            │
│ → old backups expire per ttl (see below)            │
└─────────────────────────────────────────┘
```

### Check schedule history

```bash
velero schedule get
velero backup get --selector velero.io/schedule-name=nightly-backup
```

---

## Backup Retention (TTL)

**Every backup has a Time To Live - after which Velero automatically deletes it (and its data in object storage) to control storage growth.**

```bash
velero backup create backend-backup --include-namespaces my-app --ttl 168h0m0s
```

**Visual:**
```
--ttl 168h0m0s   = 7 days
--ttl 720h0m0s   = 30 days
--ttl 8760h0m0s  = 365 days

Default TTL if unset: 720h (30 days)

Retention Strategy Example:
┌────────────────────────────────────┐
│ Hourly backups    → ttl: 24h           │  (short-term, frequent)
│ Nightly backups    → ttl: 720h (30d)     │  (medium-term)
│ Weekly backups       → ttl: 8760h (1yr)    │  (long-term/compliance)
└────────────────────────────────────┘

⚠️ Deleting a Backup object also deletes the underlying
   data in object storage AND any associated volume
   snapshots - it's a real delete, not just metadata cleanup.
```

---

## Restoring a Backup

```bash
velero restore create --from-backup backend-backup
velero restore get
velero restore describe backend-backup-20260714100000 --details
```

**Visual:**
```
velero restore create --from-backup <name>
         │
         ▼
┌─────────────────────────────────┐
│ 1. Download resources.json from     │
│    object storage                     │
│ 2. Recreate K8s API objects            │
│    (Deployments, Services, etc.)          │
│ 3. Restore PV data (CSI snapshot            │
│    or File System Backup restore)             │
│ 4. Mark Restore CRD phase: Completed              │
└─────────────────────────────────┘

Result: namespace/cluster state matches the backup
        point-in-time, including PV data
```

### Restore only part of a backup

```bash
velero restore create --from-backup backend-backup \
  --include-resources deployments,services \
  --exclude-resources persistentvolumeclaims
```

---

## Restore Namespace Mapping

**Restore into a DIFFERENT namespace than the one backed up - useful for cloning environments (e.g. prod → staging).**

```bash
velero restore create --from-backup backend-backup \
  --namespace-mappings my-app:my-app-staging
```

**Visual:**
```
Backup taken from:       Restored into:
namespace: my-app    →   namespace: my-app-staging

Use case: clone production data into a staging namespace
for testing, without touching the original production
namespace at all.
```

---

## Backup Hooks

**Run a command inside a pod BEFORE or AFTER backing it up - typically used to flush/quiesce a database for consistency.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-0
  annotations:
    pre.hook.backup.velero.io/container: database
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "pg_dump -U postgres > /backup/dump.sql"]'
    post.hook.backup.velero.io/container: database
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "rm /backup/dump.sql"]'
```

**Visual:**
```
Backup Timeline:
┌────────────────────────────────────────┐
│ 1. pre-hook runs   → pg_dump/flush cache   │
│ 2. actual backup captured (consistent!)      │
│ 3. post-hook runs  → cleanup temp files         │
└────────────────────────────────────────┘

Without hooks: risk of backing up a database mid-write,
               producing a corrupted/inconsistent snapshot
With hooks:    guaranteed consistent point-in-time capture
```

---

## Restore Hooks

**Run a command inside a pod AFTER it's restored - typically used to run a database restore/migration step.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-0
  annotations:
    post.hook.restore.velero.io/container: database
    post.hook.restore.velero.io/command: '["/bin/bash", "-c", "psql -U postgres < /backup/dump.sql"]'
    post.hook.restore.velero.io/wait-timeout: "5m"
    post.hook.restore.velero.io/exec-timeout: "1m"
```

**Visual:**
```
Restore Timeline:
┌────────────────────────────────────────┐
│ 1. Pod recreated from backup manifest       │
│ 2. Pod becomes Running                        │
│ 3. post-restore hook runs → psql restore dump  │
└────────────────────────────────────────┘

wait-timeout  → how long to wait for the pod to be ready
                before giving up on running the hook
exec-timeout  → how long the hook command itself is allowed to run
```

---

## Visual Summary

```
1. One-off backup       → velero backup create <name> --include-namespaces ns
2. Scoped backup          → add --include-resources / --selector filters
3. Recurring backup         → velero schedule create <name> --schedule="cron"
4. Retention                  → --ttl controls automatic expiry/cleanup
5. Restore                      → velero restore create --from-backup <name>
6. Clone into new namespace       → --namespace-mappings old:new
7. Consistency                      → pre/post backup hooks (flush DB)
8. Post-restore automation             → post-restore hooks (reload data)
```

---

This guide covers Velero's core backup and restore workflow - creation, scheduling, retention, namespace mapping, and hooks for consistency, with visual representations of each command.