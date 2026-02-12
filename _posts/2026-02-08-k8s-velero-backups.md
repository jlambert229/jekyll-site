---
title: "Kubernetes Cluster Backups with Velero"
subtitle: "Disaster recovery for your homelab K8s cluster. Backup everything - resources, volumes, configs. Restore to a new cluster in minutes."
date: 2026-02-08
tags:
  - kubernetes
  - homelab
  - velero
  - backup
  - disaster-recovery
  - helm
categories:
  - Kubernetes Homelab
readtime: true
---
Cluster-level disaster recovery for Kubernetes. Backup all resources, persistent volumes, and configs. Restore an entire namespace (or the whole cluster) to a new environment in minutes.
<!--more-->

> *"My cluster's etcd is corrupted. Do I have backups?"* - Questions you don't want to ask at 2am

![This is fine - Disaster recovery at 2am](/assets/img/memes/this-is-fine.jpeg)

---

## Why Velero

I had per-app backups (Sonarr SQLite, Radarr DB) but no cluster-level disaster recovery. One bad Helm upgrade - `helm upgrade metallb` with wrong IP pool - cascaded. MetalLB stopped assigning IPs. Traefik had no external IP. Plex, Sonarr, Overseerr: all unreachable. Spent four hours manually recreating the pool, ingress rules, and re-allocating LoadBalancer IPs from Git history. *"We're going to need a bigger boat."* - Jaws. Or in your case: better backups. Per-app wasn't enough.

Velero gives you:
- **Full namespace backups** - All resources (Deployments, Services, PVCs, Secrets, ConfigMaps)
- **Persistent volume snapshots** - Actual data, not just resource definitions
- **Scheduled backups** - Daily/weekly cron-like automation
- **Cross-cluster restore** - Rebuild on new hardware
- **Selective recovery** - Restore one app or the whole cluster

This complements [per-app backups](/2026-02-08-k8s-media-stack/#backups). Velero captures the Kubernetes layer (manifests, volumes). App backups capture internal state (databases, configs).

> *"Backups are worthless until you've restored from them. Test restores before you need them."* - Disaster recovery 101

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Velero (velero namespace)                                   │
│       ↓                                                      │
│  Scheduled Backups:                                          │
│   • media-daily (all resources + PVCs)                       │
│   • cluster-weekly (full cluster state)                      │
│       ↓                                                      │
│  Storage:                                                    │
│   • NFS backend (Synology: /volume1/nfs01/velero-backups)  │
│   • Restic for volume snapshots (filesystem-level)          │
│       ↓                                                      │
│  Restore:                                                    │
│   • Same cluster (rollback bad upgrades)                    │
│   • New cluster (disaster recovery)                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Deployment Repo

Full source: [k8s-velero-backups on GitHub](https://github.com/YOUR-USERNAME/k8s-velero-backups)

```
k8s-velero-backups/
├── values.yaml              # Velero Helm values
├── backup-schedules/        # CronJob-style backup definitions
│   ├── media-daily.yaml
│   └── cluster-weekly.yaml
├── deploy.sh                # Automated deployment
├── restore.sh               # Interactive restore script
└── verify-backup.sh         # Test backup integrity
```

---

## Storage Backend

Velero supports S3, GCS, Azure Blob, and filesystem targets. For homelab, NFS is simplest.

### NFS Setup on Synology

SSH to your NAS and create the backup directory:

```bash
ssh youruser@192.168.1.10
sudo mkdir -p /volume1/nfs01/velero-backups
sudo chown -R nobody:nogroup /volume1/nfs01/velero-backups
sudo chmod 755 /volume1/nfs01/velero-backups
```

Verify NFS export in DSM: Control Panel → Shared Folder → `nfs01` → Edit → NFS Permissions

Ensure your K8s subnet (`192.168.1.0/24`) has read/write access.

---

## Helm Values

```yaml
# values.yaml
image:
  repository: velero/velero
  tag: v1.14.1

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.10.1
    volumeMounts:
      - mountPath: /target
        name: plugins

configuration:
  # NFS storage via S3 API (MinIO running on NFS)
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: velero
      config:
        region: minio
        s3ForcePathStyle: "true"
        s3Url: http://minio.velero.svc.cluster.local:9000
        publicUrl: http://minio.velero.svc.cluster.local:9000

  volumeSnapshotLocation:
    - name: default
      provider: aws
      config:
        region: minio

  # Use Restic for filesystem-level PVC backups
  uploaderType: restic
  defaultVolumesToFsBackup: true

  # Namespaces to include in cluster backups
  backupSyncPeriod: 1h
  restoreOnlyMode: false

# MinIO for S3-compatible NFS backend
deployNodeAgent: true

nodeAgent:
  podVolumePath: /var/lib/kubelet/pods
  privileged: false
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

credentials:
  useSecret: true
  existingSecret: velero-credentials

# Schedules (defined separately as CRDs)
schedules: {}

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

rbac:
  create: true

serviceAccount:
  server:
    create: true
```

**Key decisions:**

- **MinIO as S3 gateway** - Wraps NFS in S3 API (Velero's native interface)
- **Restic for volumes** - Filesystem-level snapshots (doesn't require CSI snapshot support)
- **Node agent** - Runs DaemonSet to access PVCs for backup

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

Velero was originally designed for cloud object storage (S3, GCS). For homelab NFS, we run MinIO as a lightweight S3-compatible shim. This keeps Velero's API clean while storing backups on your NAS.

</div>
</div>

---

## Deploy

### 1. Install MinIO (S3 Backend)

MinIO provides the S3 API that Velero expects, backed by NFS.

Create `minio-values.yaml`:

```yaml
mode: standalone

replicas: 1

persistence:
  enabled: true
  storageClass: nfs-appdata
  size: 50Gi

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

service:
  type: ClusterIP
  port: 9000

consoleService:
  enabled: true
  port: 9001

buckets:
  - name: velero
    policy: none
    purge: false

users:
  - accessKey: velero
    secretKey: velero-secret-key
    policy: readwrite

ingress:
  enabled: true
  ingressClassName: traefik
  hosts:
    - minio.media.lan
  tls: []
```

Deploy:

```bash
helm repo add minio https://charts.min.io/
helm repo update

kubectl create namespace velero

helm upgrade --install minio minio/minio \
    -n velero -f minio-values.yaml --wait
```

Verify:

```bash
kubectl get pods -n velero
kubectl get svc -n velero
```

Access MinIO console: `http://minio.media.lan` (user: `velero`, password: `velero-secret-key`)

### 2. Create Velero Credentials Secret

```bash
cat <<EOF > credentials-velero
[default]
aws_access_key_id = velero
aws_secret_access_key = velero-secret-key
EOF

kubectl create secret generic velero-credentials \
    -n velero \
    --from-file=cloud=credentials-velero

rm credentials-velero
```

### 3. Install Velero

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

helm upgrade --install velero vmware-tanzu/velero \
    -n velero -f values.yaml --wait
```

Verify:

```bash
kubectl get pods -n velero
kubectl logs -n velero -l app.kubernetes.io/name=velero
```

You should see: `"Backup storage location is valid"`

---

## Backup Schedules

Create scheduled backups using Velero's `Schedule` CRD (like CronJobs for backups).

### Daily Media Namespace Backup

```yaml
# backup-schedules/media-daily.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: media-daily
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 2am daily
  template:
    includedNamespaces:
      - media
    includedResources:
      - '*'
    defaultVolumesToFsBackup: true
    storageLocation: default
    ttl: 168h  # Keep 7 days
```

Apply:

```bash
kubectl apply -f backup-schedules/media-daily.yaml
```

### Weekly Full Cluster Backup

```yaml
# backup-schedules/cluster-weekly.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: cluster-weekly
  namespace: velero
spec:
  schedule: "0 3 * * 0"  # 3am Sunday
  template:
    includedNamespaces:
      - '*'
    excludedNamespaces:
      - kube-system
      - kube-public
      - kube-node-lease
    includedResources:
      - '*'
    defaultVolumesToFsBackup: true
    storageLocation: default
    ttl: 720h  # Keep 30 days
```

Apply:

```bash
kubectl apply -f backup-schedules/cluster-weekly.yaml
```

**Verify schedules:**

```bash
velero schedule get
velero backup get
```

---

## Manual Backups

Trigger an immediate backup:

```bash
# Backup entire media namespace
velero backup create media-manual \
    --include-namespaces media \
    --default-volumes-to-fs-backup \
    --wait

# Backup single app (Sonarr)
velero backup create sonarr-manual \
    --include-namespaces media \
    --selector app.kubernetes.io/name=sonarr \
    --default-volumes-to-fs-backup \
    --wait

# Full cluster backup
velero backup create cluster-manual \
    --exclude-namespaces kube-system,kube-public,kube-node-lease \
    --default-volumes-to-fs-backup \
    --wait
```

Check status:

```bash
velero backup describe media-manual
velero backup logs media-manual
```

---

## Restore

### Restore Entire Namespace

**Scenario:** Bad Helm upgrade broke the media namespace.

```bash
# 1. Delete broken namespace (optional but cleaner)
kubectl delete namespace media

# 2. Restore from latest backup
velero restore create media-restore-$(date +%s) \
    --from-backup media-daily-20260208020000 \
    --wait

# 3. Verify
kubectl get all -n media
```

### Restore Single App

**Scenario:** Sonarr's database corrupted. Restore just Sonarr.

```bash
# 1. Scale down Sonarr
kubectl scale -n media deploy/sonarr --replicas=0

# 2. Restore Sonarr resources
velero restore create sonarr-restore-$(date +%s) \
    --from-backup media-daily-20260208020000 \
    --include-resources deployment,service,ingress,pvc,secret,configmap \
    --selector app.kubernetes.io/name=sonarr \
    --wait

# 3. Verify
kubectl get pods -n media -l app.kubernetes.io/name=sonarr
```

### Disaster Recovery (New Cluster)

**Scenario:** Entire cluster lost (hardware failure, etcd corruption).

1. **Build new cluster** - Use [k8s-deploy Terraform repo](/2026-02-08-k8s-talos-proxmox-deploy/)
2. **Install foundation** - MetalLB, Traefik, NFS CSI, Velero (same as original)
3. **Point Velero at existing backups:**

```bash
# MinIO already deployed with same NFS backend
# Velero sees existing backups automatically
velero backup get
```

4. **Restore cluster state:**

```bash
velero restore create full-restore-$(date +%s) \
    --from-backup cluster-weekly-20260202030000 \
    --wait
```

5. **Verify all namespaces:**

```bash
kubectl get namespaces
kubectl get all -n media
kubectl get all -n monitoring
```

---

## Backup Verification

> *"Hope is not a restore strategy. Verify before the incident."* - Every SRE who learned the hard way

Don't trust backups you haven't tested. The repo includes `verify-backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

BACKUP_NAME="${1:-}"

if [[ -z "$BACKUP_NAME" ]]; then
    echo "Usage: $0 <backup-name>"
    echo ""
    echo "Available backups:"
    velero backup get
    exit 1
fi

echo "Verifying backup: $BACKUP_NAME"

# Check backup completed successfully
STATUS=$(velero backup describe "$BACKUP_NAME" --details | grep -i phase | awk '{print $2}')

if [[ "$STATUS" != "Completed" ]]; then
    echo "❌ Backup status: $STATUS"
    exit 1
fi

echo "✅ Backup status: Completed"

# Check for errors
ERRORS=$(velero backup describe "$BACKUP_NAME" --details | grep -i errors | awk '{print $2}')

if [[ "$ERRORS" != "0" ]]; then
    echo "⚠️  Backup has $ERRORS errors:"
    velero backup logs "$BACKUP_NAME" | grep -i error
    exit 1
fi

echo "✅ No errors"

# Verify volumes backed up
VOLUMES=$(velero backup describe "$BACKUP_NAME" --details | grep -A 20 "Restic Backups" | grep "Completed: " | awk '{print $2}')

echo "✅ Volumes backed up: $VOLUMES"

# Check backup size in MinIO
echo ""
echo "Backup stored in MinIO (velero bucket)"
```

Run monthly:

```bash
./verify-backup.sh media-daily-20260208020000
```

---

## Troubleshooting

### Backup Stuck in Progress

**Symptom:** Backup never completes.

```bash
velero backup describe <backup-name>
```

**Common causes:**

- **Restic timeout** - Large volumes take time. Increase timeout:
  ```yaml
  # values.yaml
  configuration:
    fsBackupTimeout: 4h  # Default 1h
  ```

- **PVC not found** - Velero can't access PVC. Check node agent pods:
  ```bash
  kubectl get pods -n velero -l name=node-agent
  kubectl logs -n velero -l name=node-agent
  ```

### Restore Fails with "Already Exists"

**Symptom:** `velero restore` fails because resources already exist.

**Fix:** Delete and retry, or use `--preserve-nodeports=false` for Services.

```bash
kubectl delete namespace media
velero restore create media-restore-$(date +%s) --from-backup <backup> --wait
```

### MinIO Connection Refused

**Symptom:** Velero logs show `connection refused` to MinIO.

**Check:**

```bash
kubectl get svc -n velero minio
kubectl logs -n velero -l app.kubernetes.io/name=velero | grep -i minio
```

**Fix:** Verify MinIO service is running and accessible:

```bash
kubectl exec -n velero deploy/velero -- wget -O- http://minio.velero.svc.cluster.local:9000
```

### Backup Storage Location Unavailable

**Symptom:** `velero backup-location get` shows `Unavailable`.

**Check:**

```bash
velero backup-location describe default
```

**Common causes:**

- **Wrong credentials** - Verify `velero-credentials` secret
- **MinIO not running** - Check `kubectl get pods -n velero`
- **Bucket doesn't exist** - Create `velero` bucket in MinIO console

---

## Resource Usage

Tested on 2-worker cluster (2 vCPU, 4 GB RAM per worker):

- **Velero server:** 50 MB RAM, <1% CPU (idle)
- **Node agent (per node):** 100 MB RAM, <5% CPU (during backup)
- **MinIO:** 200 MB RAM, <5% CPU

**Backup times:**
- Media namespace (8 apps, 40 GB PVCs): ~15 minutes
- Full cluster (3 namespaces, 60 GB total): ~25 minutes

**Storage:**
- Daily media backups: ~8 GB each (with compression)
- 7-day retention: ~56 GB
- Weekly full cluster: ~15 GB each
- 30-day retention: ~120 GB total

Provision 200 GB on your NAS for Velero backups.

---

## What I Learned

### 1. Test Restores, Not Just Backups

> *"The only backup that matters is the one you've successfully restored from."* - Chaos engineering corollary

I ran Velero backups for four months. Never tested a restore. Then a bad Helm upgrade (`helm upgrade traefik` with a broken values file) nuked the media namespace - Traefik's CRDs got deleted, cascade deleted a bunch of Ingresses. I ran `velero restore create`. It failed. Missing CRDs. The Traefik IngressRoute CRDs weren't in the backup scope; Velero backs up resources, not necessarily the CRDs that define them. I spent two hours manually recreating ingress resources from Git history while Plex stayed down. My family's weekend movie plans were collateral damage. I now schedule a quarterly "chaos day" where I delete a namespace and restore. Found two more issues: PVC restore failed because the StorageClass had been renamed, and Secrets with immutable fields broke. I fix these proactively now. Don't learn this the way I did.

### 2. Separate App-Level and Cluster-Level Backups

Velero backs up Kubernetes state. [Per-app backups](/2026-02-08-k8s-media-stack/#backups) back up internal databases. You need both.

True story: I restored Sonarr from Velero. The Deployment, Service, PVC - all back. Sonarr started. Opened the UI. Zero shows. Zero history. The PVC had the right permissions but the SQLite database was from *before* I'd added anything. Velero restored the empty volume. App-level backups would have restored the actual database. I had to re-add 200+ series. I am still finding things I forgot to re-add. Both layers. Always both.

### 3. MinIO Adds Latency but Simplifies Ops

Considered backing up directly to NFS (Velero's filesystem plugin). MinIO adds a hop, but: S3 API is Velero's native interface (fewer bugs), MinIO console makes browsing backups easy, and I can switch to Backblaze B2 later with zero config change. The 50 MB of extra memory is worth it. I tried the filesystem plugin once. Velero's NFS support was... let's say "enthusiastically documented but sparingly implemented." MinIO just works.

### 4. Backup Everything, Restore Selectively

Full cluster backups sound expensive. They're 15 GB. I back up everything weekly. One time I ran `kubectl delete namespace media` while tired. I'd meant to delete `media-test` or `default` - I was cleaning up. Autocomplete betrayed me. I stared at the terminal. Plex, Sonarr, Radarr, all of it - gone. The backup from 2am had everything. `velero restore create media-restore --from-backup media-daily-20260208020000 --wait`. Twenty minutes. Crisis averted. I added `media` to my "never delete" list. And stopped using `kubectl` when sleepy. Having options reduces stress. Having *no* option leads to a very long night.

### 5. Retention Policies Save Disk Space

First month, I kept every backup forever. "Storage is cheap." Hit 500 GB. The NAS sent me a warning. Then my spouse asked why the homelab backup folder was larger than our entire photo library. I had no defensible answer. Set TTLs: daily 7 days, weekly 30 days, monthly 1 year. Now 200 GB. Auto-pruning means I never have to think about it. "I'll clean this up later" is a lie we tell ourselves. Make the system do it.

---

## What's Next

You have disaster recovery for your Kubernetes cluster. Restore individual apps or rebuild from scratch.

**Optional enhancements:**

- **Off-site backups** - Sync MinIO to Backblaze B2 or AWS S3 for geographic redundancy
- **Pre/post hooks** - Quiesce databases before backup (flush writes, snapshot consistency)
- **Monitoring integration** - Alert on failed backups via [Uptime Kuma](/2026-02-08-k8s-uptime-kuma/)
- **Immutable backups** - Enable S3 object lock to prevent ransomware deletion

The core setup is production-ready. Sleep better knowing you can rebuild in minutes.

---

## References

- [Velero Documentation](https://velero.io/docs/)
- [Velero Helm Chart](https://github.com/vmware-tanzu/helm-charts/tree/main/charts/velero)
- [MinIO Helm Chart](https://github.com/minio/minio/tree/master/helm/minio)
- [Restic Integration](https://velero.io/docs/main/file-system-backup/)
- [Disaster Recovery Best Practices](https://velero.io/docs/main/disaster-case/)
