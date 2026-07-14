# Velero in Production - RBAC, Monitoring, Upgrades, Troubleshooting

Operational patterns for running Velero reliably as the backbone of your DR strategy, with visual diagrams.

## Table of Contents
- [RBAC and Least Privilege](#rbac-and-least-privilege)
- [Monitoring Backup Health](#monitoring-backup-health)
- [Alerting on Backup Failures](#alerting-on-backup-failures)
- [Backup Strategy Design](#backup-strategy-design)
- [Resource Planning](#resource-planning)
- [Upgrading Velero](#upgrading-velero)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [CI/CD Integration Pattern](#cicd-integration-pattern)
- [Uninstalling Velero](#uninstalling-velero)

---

## RBAC and Least Privilege

**The Velero server's cloud credentials are extremely powerful (can read/write your entire object storage bucket, and often has broad cluster-read permissions) - lock this down.**

**Visual:**
```
Cloud IAM Policy (AWS example) - scoped to ONLY what Velero needs:
┌──────────────────────────────────────────┐
│ s3:GetObject, s3:PutObject, s3:DeleteObject  │
│  → only on arn:aws:s3:::my-velero-backups/*      │
│ ec2:CreateSnapshot, ec2:DeleteSnapshot,             │
│  ec2:DescribeSnapshots                                │
│  → only for EBS snapshot operations                     │
└──────────────────────────────────────────┘

⚠️ Do NOT grant the Velero IAM role/service account
   broader S3 access (e.g. s3:*) or admin-level cluster
   permissions "just in case" - it's a backup tool holding
   a copy of everything, so its own blast radius if
   compromised is enormous.
```

**Kubernetes RBAC:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: velero-restricted
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch", "create"]   # avoid delete/update
                                                   # cluster-wide if possible
```

---

## Monitoring Backup Health

```bash
velero backup get
velero backup describe <name>
```

**Visual:**
```
Backup Phases:
New          → just created, not yet processed
InProgress   → actively backing up
Completed    → fully successful
PartiallyFailed → some items backed up, some failed (⚠️ investigate!)
Failed       → backup did not succeed

┌────────────────────────────────────────┐
│ Daily health check script should verify:    │
│  1. Last scheduled backup exists              │
│  2. Its phase is "Completed" (not               │
│     PartiallyFailed or Failed)                    │
│  3. Its age is within expected schedule window       │
└────────────────────────────────────────┘
```

### Prometheus metrics

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: velero
  namespace: velero
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: velero
  endpoints:
    - port: monitoring
```

**Visual:**
```
Key metrics exposed by Velero:
velero_backup_success_total
velero_backup_failure_total
velero_backup_duration_seconds
velero_backup_last_successful_timestamp

Alert rule example:
"page on-call if velero_backup_last_successful_timestamp
 is older than 26 hours" (for a nightly schedule)
```

---

## Alerting on Backup Failures

**A failed backup is a silent risk - no one notices until they need to restore and discover there's nothing usable.**

**Visual:**
```
┌──────────────────────────────────────────────┐
│ Alertmanager Rule                                  │
├──────────────────────────────────────────────┤
│ alert: VeleroBackupFailed                           │
│ expr: velero_backup_failure_total > 0                 │
│ for: 5m                                                  │
│ annotations:                                                │
│   summary: "Velero backup failed - DR posture degraded"       │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ alert: VeleroBackupStale                                │
│ expr: time() - velero_backup_last_successful_timestamp    │
│        > 93600   # 26 hours                                  │
│ annotations:                                                    │
│   summary: "No successful backup in over 26h"                     │
└──────────────────────────────────────────────┘

Treat backup failures with the SAME urgency as a
production outage alert - by the time you need the
backup, it's too late to fix a silent failure.
```

---

## Backup Strategy Design

**Visual:**
```
Layered Retention Strategy (common production pattern):

┌────────────────────────────────────────┐
│ Hourly  → ttl: 48h    (fast rollback for    │
│                         recent mistakes)      │
│ Daily    → ttl: 720h (30d) (general safety     │
│                              net)                 │
│ Weekly    → ttl: 8760h (1yr) (compliance/          │
│                                long-term audit)      │
└────────────────────────────────────────┘

3 separate Schedule objects, one per cadence:
velero schedule create hourly-backup --schedule="0 * * * *" --ttl 48h0m0s
velero schedule create daily-backup --schedule="0 2 * * *" --ttl 720h0m0s
velero schedule create weekly-backup --schedule="0 3 * * 0" --ttl 8760h0m0s

Also consider: separate schedules per namespace/criticality
tier, so a critical payments namespace can have tighter
RPO (recovery point objective) than a low-priority dev namespace.
```

---

## Resource Planning

**Visual:**
```
velero (Deployment):
┌───────────────────────────────┐
│ CPU request:    500m               │
│ Memory request: 128Mi               │
│ (spikes during large backup/restore   │
│  operations - monitor and tune)          │
└───────────────────────────────┘

node-agent (DaemonSet, per node):
┌───────────────────────────────┐
│ CPU request:    500m               │
│ Memory request: 512Mi               │
│ (File System Backup is CPU/memory     │
│  intensive - chunking + dedup work       │
│  happens here during backup/restore)        │
└───────────────────────────────┘

Object Storage cost:
Plan for bucket growth based on retention policy ×
number of namespaces × average PV size - File System
Backup deduplication helps, but budget realistically.
```

---

## Upgrading Velero

```bash
kubectl label deployment/velero -n velero app.kubernetes.io/name=velero --overwrite
velero install --crds-only --dry-run -o yaml | kubectl apply -f -

kubectl set image deployment/velero -n velero \
  velero=velero/velero:v1.14.0

kubectl set image daemonset/node-agent -n velero \
  node-agent=velero/velero:v1.14.0
```

**Visual:**
```
Upgrade Order (important!):
1. Read release notes for CRD/breaking changes
2. Update CRDs first
   velero install --crds-only --dry-run -o yaml | kubectl apply -f -
3. Update the velero Deployment image
4. Update the node-agent DaemonSet image
5. Verify with a test backup + restore before relying
   on the new version for real recovery

⚠️ Never skip multiple major versions at once - upgrade
   incrementally and check the compatibility matrix,
   since Backup/Restore CRD schemas do evolve.
```

---

## Troubleshooting Checklist

```bash
# 1. Is the Velero server healthy?
kubectl get pods -n velero
kubectl logs deployment/velero -n velero

# 2. Is the backup location reachable?
velero backup-location get

# 3. What exactly failed in a backup?
velero backup describe <name> --details
velero backup logs <name>

# 4. Is node-agent healthy on the relevant node?
kubectl get pods -n velero -o wide | grep node-agent
kubectl logs <node-agent-pod> -n velero

# 5. What failed in a restore?
velero restore describe <name> --details
velero restore logs <name>
```

**Visual:**
```
Common Issue → Root Cause → Fix
──────────────────────────────────────────────────────────
Backup phase:            → cloud credentials expired/    → rotate secret,
Failed                      insufficient IAM permissions    velero delete +
                                                              recreate BSL

PartiallyFailed backup     → pre-hook command failed on    → check hook
                              specific pod (e.g. pg_dump      command syntax,
                              exit code non-zero)               increase timeout

Restore stuck              → StorageClass on destination    → add change-
                              cluster doesn't exist             storage-class
                                                                 ConfigMap mapping

node-agent CrashLoop         → insufficient privileged        → check node
                                access / PSP-PSA blocking         security policy
                                                                    allows it

Volume data missing after     → File System Backup annotation  → confirm
restore                          excluded the volume, or CSI       backup.velero.io
                                  snapshot class missing              annotations and
                                                                        VolumeSnapshotClass
```

---

## CI/CD Integration Pattern

**Trigger an on-demand backup as a pipeline stage before risky deployments - not just relying on scheduled backups.**

```bash
BACKUP_NAME="pre-deploy-$(date +%s)"
velero backup create "$BACKUP_NAME" --include-namespaces my-app --wait

STATUS=$(velero backup get "$BACKUP_NAME" -o json | jq -r '.status.phase')
if [ "$STATUS" != "Completed" ]; then
  echo "Pre-deploy backup failed - aborting deployment"
  exit 1
fi

echo "Backup $BACKUP_NAME confirmed, proceeding with deploy"
```

**Visual:**
```
Pipeline: deploy-backend
┌──────────────────────────────────────────────┐
│ 1. Pre-deploy backup (velero backup create --wait) │
│ 2. Gate: backup phase must be Completed                │
│ 3. Deploy new version                                     │
│ 4. Smoke test                                                │
│ 5. If smoke test fails:                                        │
│    velero restore create --from-backup pre-deploy-<ts>          │
└──────────────────────────────────────────────┘
```

---

## Uninstalling Velero

```bash
kubectl delete namespace velero
kubectl delete crds -l component=velero
```

**Visual:**
```
⚠️ Deleting the velero namespace does NOT delete your
   backups in object storage - they remain safely in
   the S3/GCS/Azure bucket. Only the in-cluster CRDs
   and controller are removed.

To also delete backup DATA (rarely what you want):
   manually empty/delete the object storage bucket
```

---

## Visual Summary

```
Production Checklist:
☑ Cloud credentials scoped to least privilege (bucket + snapshot only)
☑ Layered backup schedules (hourly/daily/weekly) with matching TTLs
☑ Prometheus metrics + Alertmanager rules on backup failure/staleness
☑ node-agent resource requests tuned for File System Backup workload
☑ Pre-deploy backups gated in CI/CD before risky changes
☑ StorageClass mapping ConfigMap ready for cross-cluster restores
☑ Quarterly DR test - actually restore into a throwaway cluster
☑ Upgrade CRDs before Deployment/DaemonSet images, never skip majors
```

---

This guide covers running Velero operationally in production - RBAC scoping, monitoring/alerting, retention strategy, upgrades, and troubleshooting, with visual representations of each pattern.