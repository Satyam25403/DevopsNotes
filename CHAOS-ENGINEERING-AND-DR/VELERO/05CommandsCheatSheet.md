# Velero Command Cheat Sheet

Quick reference for all Velero commands covered across the guides.

---

## Install & Setup

```bash
brew install velero
velero version --client-only

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero \
  --use-node-agent

kubectl get pods -n velero
velero backup-location get
velero snapshot-location get
```

## Backups

```bash
velero backup create my-backup --include-namespaces my-app
velero backup create my-backup \
  --include-namespaces my-app \
  --include-resources deployments,services,persistentvolumeclaims \
  --exclude-resources events \
  --selector app=database \
  --ttl 168h0m0s

velero backup get
velero backup describe my-backup --details
velero backup logs my-backup
velero backup delete my-backup
```

## Scheduled Backups

```bash
velero schedule create nightly-backup --schedule="0 2 * * *" --ttl 720h0m0s
velero schedule get
velero backup get --selector velero.io/schedule-name=nightly-backup
velero schedule delete nightly-backup
```

## Restores

```bash
velero restore create --from-backup my-backup
velero restore create --from-backup my-backup \
  --include-resources deployments,services \
  --exclude-resources persistentvolumeclaims
velero restore create --from-backup my-backup \
  --namespace-mappings my-app:my-app-staging

velero restore get
velero restore describe <restore-name> --details
velero restore logs <restore-name>
```

## Hooks (pod annotations)

```yaml
pre.hook.backup.velero.io/container: database
pre.hook.backup.velero.io/command: '["/bin/bash","-c","pg_dump ..."]'
post.hook.backup.velero.io/command: '["/bin/bash","-c","cleanup..."]'
post.hook.restore.velero.io/command: '["/bin/bash","-c","psql restore..."]'
post.hook.restore.velero.io/wait-timeout: "5m"
```

## Volume Backup Annotations

```yaml
backup.velero.io/backup-volumes: "data-volume"
backup.velero.io/backup-volumes-excludes: "cache-volume,tmp-volume"
```

## StorageClass Mapping (migration)

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

## Data Mover (cross-cloud CSI)

```yaml
spec:
  snapshotMoveData: true
```

## Upgrade & Uninstall

```bash
velero install --crds-only --dry-run -o yaml | kubectl apply -f -
kubectl set image deployment/velero -n velero velero=velero/velero:v1.14.0
kubectl set image daemonset/node-agent -n velero node-agent=velero/velero:v1.14.0

kubectl delete namespace velero
kubectl delete crds -l component=velero
```

---

## Command Quick Index

| Command | Purpose |
|---|---|
| `velero install` | Install Velero server + node-agent into a cluster |
| `velero backup create` | Create an on-demand backup |
| `velero schedule create` | Automate recurring backups |
| `velero backup get / describe / logs` | Inspect backup status and details |
| `velero restore create --from-backup` | Restore resources (and optionally PV data) |
| `velero backup-location get` | Check object storage connectivity |
| `velero snapshot-location get` | Check volume snapshot config |

---

This cheat sheet summarizes all Velero commands from the basics, backup/restore, volume snapshots/file backup, migration/DR, and production operations guides.