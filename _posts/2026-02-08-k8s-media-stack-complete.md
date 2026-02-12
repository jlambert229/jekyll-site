---
title: "Kubernetes Media Stack: Plex, Sonarr, Radarr, and Friends"
subtitle: "A complete automated media stack on Kubernetes with TLS, monitoring, and production-grade configurations."
date: 2026-02-08
tags:
  - kubernetes
  - talos
  - homelab
  - plex
  - media
  - automation
categories:
  - Kubernetes Homelab
readtime: true
---
The complete build: Plex, Sonarr, Radarr, and the entire automation stack on Kubernetes. Production patterns, homelab context, actual lessons from running this for 18 months.
<!--more-->

## Background

Docker Compose worked fine for two years. Then the host died.

I had backups of the config files. I had documentation (somewhere). I had a vague memory of how things connected. What I didn't have was a clean way to rebuild without spending an entire weekend re-learning my own setup.

That's when I moved to Kubernetes. Not because Kubernetes is simpler - it's not. Because Kubernetes is *reproducible*. Everything lives in YAML. YAML lives in Git. Lose the cluster? `git clone && kubectl apply`. Fifteen minutes later you're back.

I've rebuilt this stack three times now:
1. First time: moving from Docker Compose (painful, took a weekend)
2. Second time: upgrading hardware (2 hours)
3. Third time: testing disaster recovery (47 minutes)

Each rebuild got faster because the config was already correct.

Source: [k8s-media-stack](https://github.com/YOUR-USERNAME/k8s-media-stack)

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Homelab, not production</div>
<div class="admonition-content" markdown="1">

Self-signed certs. Some hardcoded values. Basic security. This runs on a home network behind NAT. Don't deploy this at work without hardening it first.

</div>
</div>

---

## The Stack

| App | Purpose |
|-----|---------|
| **Plex** | Media server for streaming |
| **Sonarr** | TV show automation |
| **Radarr** | Movie automation |
| **SABnzbd** | Usenet downloads |
| **Overseerr** | User request management |
| **Prowlarr** | Indexer management |
| **Bazarr** | Subtitle downloads |
| **Tautulli** | Plex statistics |
| **Homepage** | Unified dashboard |

**Infrastructure:**

| Component | Purpose |
|-----------|---------|
| **Traefik** | Ingress + TLS termination |
| **cert-manager** | Automated cert issuance |
| **MetalLB** | LoadBalancer for bare metal |
| **NFS CSI Driver** | Network storage |
| **local-path** | Fast local storage |

**Monitoring:**

| Component | Purpose |
|-----------|---------|
| **Grafana** | Dashboards and visualization |
| **Prometheus** | Metrics collection |
| **Alertmanager** | Email notifications |

---

## The Storage Problem

First iteration: everything on NFS. One shared volume, all the data. Clean. Simple. Elegant.

Also: completely unusable.

Sonarr took 45 seconds to load the series list. Radarr was worse - over a minute to display the movie library. The web UIs felt broken. Logs showed thousands of SQLite queries timing out.

The problem: these apps hammer SQLite with small random reads. NFS adds 5-10ms latency to every operation. One query becomes 100ms. A page that needs 200 queries becomes 20 seconds of waiting.

**The fix: split storage by access pattern**

Media files (movies, TV shows) stay on NFS:
- Large sequential reads (streaming)
- Shared across multiple apps
- Network latency doesn't matter for 20GB video files

App configs (SQLite databases) move to local SSDs:
- Small random reads (metadata queries)
- Performance critical
- Each app pinned to one node (no concurrent access needed)

After the move: page loads dropped to 2 seconds. The apps became responsive. This isn't an optimization - it's a requirement.

Database sizes: check before deciding. Most of these apps use under 500MB:

```bash
kubectl exec -n media deploy/sonarr -- du -sh /config  # Usually ~300MB
```

Under 2GB? Local storage. Over 2GB or needs sharing? Then consider the performance cost.

---

## Foundation: TLS and Storage

### TLS Certificates

Self-signed CA for `*.media.lan`. Not for production, perfect for homelab. Generate once, trust on your devices, forget about it:

```bash
# Generate CA
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 \
  -nodes -keyout ca.key -out ca.crt \
  -subj "/CN=Media Stack CA" \
  -addext "subjectAltName=DNS:*.media.lan"

# Create secret
kubectl create secret tls media-lan-ca \
  --cert=ca.crt --key=ca.key -n cert-manager

# ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: media-lan-ca
spec:
  ca:
    secretName: media-lan-ca
EOF
```

cert-manager now issues certs automatically for any Ingress in the `media` namespace. Add the annotation, get a cert.

**Production note:** Let's Encrypt for internet-facing services. Vault or enterprise CA for corporate. Self-signed works here because I control all the clients and can add the CA to their trust stores.

### Storage Classes

```yaml
# NFS storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-appdata
provisioner: nfs.csi.k8s.io
parameters:
  server: <NAS_IP>               # e.g. 192.168.1.100
  share: /volume1/nfs01/data
mountOptions:
  - vers=3
  - soft
  - intr

---
# Local storage (Rancher local-path-provisioner)
# Already installed - just use storageClassName: local-path
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

The `soft,intr` mount options mean NFS timeouts return errors instead of hanging forever. Without this, when your NAS reboots, every pod with an NFS mount hangs indefinitely. You can't even kill them. Learn from my pain: use `soft,intr`.

</div>
</div>

---

## App Configuration Pattern

Every app uses `bjw-s/app-template` for consistency. Here's the pattern:

```yaml
# Example: Sonarr
controllers:
  sonarr:
    containers:
      app:
        image:
          repository: ghcr.io/onedr0p/sonarr
          tag: 4.0.11
        
        # TCP probes more reliable than HTTP
        probes:
          startup:
            enabled: true
            spec:
              tcpSocket:
                port: 8989
              failureThreshold: 60  # 5 minutes to start
              periodSeconds: 5
          
          liveness:
            enabled: true
            spec:
              tcpSocket:
                port: 8989
              periodSeconds: 30
              failureThreshold: 10  # 5 minutes before restart
          
          readiness:
            enabled: true
            spec:
              tcpSocket:
                port: 8989
              periodSeconds: 10
        
        # Prevent memory leaks from killing the node
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: "2"
            memory: 2Gi

service:
  app:
    controller: sonarr
    ports:
      http:
        port: 8989

ingress:
  app:
    className: traefik
    annotations:
      cert-manager.io/cluster-issuer: media-lan-ca
    hosts:
      - host: sonarr.media.lan
        paths:
          - path: /
            service:
              identifier: app
              port: http
    tls:
      - hosts:
          - sonarr.media.lan
        secretName: sonarr-tls

persistence:
  config:
    type: persistentVolumeClaim
    accessMode: ReadWriteOnce
    size: 2Gi
    storageClass: local-path  # Fast local storage
    globalMounts:
      - path: /config
  
  # Cache posters locally instead of hitting NFS
  mediacover:
    type: emptyDir
    globalMounts:
      - path: /config/MediaCover
  
  # Shared media storage
  data:
    type: persistentVolumeClaim
    existingClaim: media-data
    globalMounts:
      - path: /data
```

The `bjw-s/app-template` chart is generic. One pattern, all eight apps. Learn it once, use it everywhere. At 2am when Plex is down, you don't want to be figuring out eight different deployment patterns.

### Critical Details

**TCP probes, not HTTP**

Sonarr kept restarting during updates. HTTP probe hit the endpoint too early, before the app was ready. Probe failed, Kubernetes killed the pod, restart loop.

Switched to TCP probes - just check if the port opens:

```yaml
tcpSocket:
  port: 8989
failureThreshold: 60  # These apps are SLOW to start
```

These aren't cloud-native apps optimized for fast startup. They're PHP/Python apps with SQLite databases. Give them time.

**Cache locally with emptyDir**

Sonarr downloaded poster images from NFS on every page load. Same 200 images, every single time. NAS was getting hammered unnecessarily.

Solution: mount emptyDir over the cache directory. Posters cache on local disk:

```yaml
mediacover:
  type: emptyDir
  globalMounts:
    - path: /config/MediaCover
```

NFS I/O dropped 60% immediately. One line of YAML.

**Resource limits aren't optional**

Plex killed my node. Twice.

First time: didn't set limits. Plex transcoded four 4K streams simultaneously, consumed all available RAM, triggered the kernel OOM killer. Random system processes died to free memory. Node became unresponsive.

Second time: set limits too high. Same result.

Third time: actually profiled resource usage, set limits based on data:

```yaml
resources:
  limits:
    memory: 4Gi  # Pod OOMs before affecting node
```

How to set limits correctly:
1. Run workload for a week
2. Check actual usage: `kubectl top pods -n media`
3. Set limits 20-30% above observed peaks

Too tight = legitimate spikes OOM the pod  
Too loose = you're back where you started  
Just right = pod dies, node survives

---

## Plex Configuration

Plex needs a LoadBalancer IP for direct access (DLNA, clients):

```yaml
service:
  app:
    controller: plex
    type: LoadBalancer  # MetalLB assigns an IP
    loadBalancerIP: <PLEX_IP>     # e.g. 192.168.1.245
    ports:
      http:
        port: 32400

persistence:
  config:
    storageClass: local-path  # Metadata DB needs fast I/O
    size: 25Gi
  
  data:
    existingClaim: media-data  # Shared NFS
    globalMounts:
      - path: /data/media
```

**Hardlinks require shared filesystem:**

This matters more than you think. When Sonarr "moves" a completed download to your library, it needs to be instant. If downloads and media are on different filesystems, it actually *copies* the file. That means:
- Your 50GB 4K movie takes 5 minutes to "move"
- You need 50GB of free space you didn't need before
- It's hitting NFS with sustained sequential writes

Hardlinks solve this. Same NFS volume, different mount paths:

```yaml
# Same NFS PVC mounted at /data in all apps
volumes:
  - name: media-data
    nfs:
      server: <NAS_IP>             # e.g. 192.168.1.100
      path: /volume1/nfs01/data

# Directory structure on NFS:
/data/
  media/
    tv/      # Sonarr final location
    movies/  # Radarr final location
  downloads/
    complete/  # SABnzbd output
```

Now Sonarr "moves" a 50GB file in under a second. It's just updating the directory entry. Same file, new path. No copy.

---

## SABnzbd Setup

Switched from qBittorrent after dealing with too many dead torrents and ratio requirements. Usenet is faster, more reliable, and doesn't care about seeding.

```yaml
persistence:
  config:
    storageClass: local-path  # Fast DB
    size: 1Gi
  
  incomplete-downloads:
    type: emptyDir  # Temp downloads on fast local disk
    sizeLimit: 100Gi
  
  data:
    existingClaim: media-data  # Final destination on NFS
    globalMounts:
      - path: /data
```

**Configure in Web UI:**

1. Add Newshosting server (Settings → Servers)
2. Set categories (Settings → Categories):
   - `tv` → `/data/downloads/complete/tv`
   - `movies` → `/data/downloads/complete/movies`
3. Enable API (Settings → General)

**Integrate with Prowlarr:**

Prowlarr is the indexer manager. Add your indexers once there, and it pushes them to Sonarr/Radarr via API. This beats manually adding the same 5 indexers to three different apps.

In Prowlarr: Settings → Apps → Add Application → Pick Sonarr or Radarr. It autodiscovers via DNS if they're in the same namespace. If not, use service DNS names: `http://sonarr.media.svc.cluster.local:8989`

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Grab API keys from Settings → General in each app. You'll need them for cross-app integrations. Store them somewhere - you'll use them again when you rebuild this in six months.

</div>
</div>

---

## Monitoring Stack

> *"If you can't measure it, you can't improve it. If you can't observe it, you can't fix it."* - Operations Mantra

You need monitoring. Not because it's best practice - because you need to know when Plex is approaching its memory limit before the pod OOMs at 10pm on a Friday while your family is watching a movie.

Deploy kube-prometheus-stack:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n media \
  -f apps/monitoring/values.yaml
```

### Key Configuration

```yaml
# apps/monitoring/values.yaml
grafana:
  persistence:
    storageClass: local-path  # SQLite needs fast I/O
    size: 5Gi
  
  ingress:
    enabled: true
    className: traefik
    hosts:
      - grafana.media.lan
    tls:
      - hosts:
          - grafana.media.lan
  
  defaultDashboardsEnabled: true  # Pre-built K8s dashboards

prometheus:
  prometheusSpec:
    retention: 7d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-appdata  # Historical data on NFS
          resources:
            requests:
              storage: 10Gi

alertmanager:
  config:
    global:
      smtp_from: 'your-email@gmail.com'
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_auth_username: 'your-email@gmail.com'
      smtp_auth_password: 'your-gmail-app-password'
      smtp_require_tls: true
    
    receivers:
      - name: 'email'
        email_configs:
          - to: 'your-email@gmail.com'
            send_resolved: true
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Gmail requires an app password for SMTP. Generate one at https://myaccount.google.com/apppasswords. Regular password won't work. Don't ask me how I know.

</div>
</div>

### Custom Dashboard

Auto-imported via ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-media-stack
  namespace: media
  labels:
    grafana_dashboard: "1"  # Grafana sidecar imports this
data:
  media-stack-overview.json: |
    { ... dashboard JSON ... }
```

**Panels:**
- CPU/Memory usage by app
- Pod status (visual health check)
- Network I/O (Plex streaming + downloads)
- Container restarts
- Storage usage

Access at: `https://grafana.media.lan`

### Alert Rules

```yaml
# monitoring/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: media-stack-alerts
  namespace: media
spec:
  groups:
    - name: media-stack
      rules:
        - alert: MediaAppPodDown
          expr: kube_pod_status_phase{namespace="media", phase="Running", pod=~"plex.*|sonarr.*|radarr.*"} == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "{{ $labels.pod }} is down"
        
        - alert: MediaStorageFull
          expr: 100 * kubelet_volume_stats_used_bytes{persistentvolumeclaim="media-data"} / kubelet_volume_stats_capacity_bytes{persistentvolumeclaim="media-data"} > 85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Media storage {{ $value | humanize }}% full"
```

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

You'll get two emails per alert: one when it fires, one when it resolves. Don't disable the resolved notifications - knowing that things fixed themselves is useful information.

</div>
</div>

---

## Homepage Dashboard

Unified view with live widgets:

```yaml
# dashboard/homepage.yaml
services:
  - Media:
      - Plex:
          href: http://<PLEX_IP>:32400/web
      - Overseerr:
          widget:
            type: overseerr
            url: http://overseerr.media.svc.cluster.local:5055
            key: <api-key>
  
  - Downloads:
      - Sonarr:
          widget:
            type: sonarr
            url: http://sonarr.media.svc.cluster.local:8989
            key: <api-key>
      - Radarr:
          widget:
            type: radarr
            url: http://radarr.media.svc.cluster.local:7878
            key: <api-key>
      - SABnzbd:
          widget:
            type: sabnzbd
            url: http://sabnzbd.media.svc.cluster.local:8080
            key: <api-key>
  
  - Monitoring:
      - Grafana:
          widget:
            type: grafana
            url: http://kube-prometheus-stack-grafana.media.svc.cluster.local
```

Access at: `https://home.media.lan`

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Grab API keys from Settings → General → API Key in each app. The widgets show real-time queue sizes and disk usage. Way better than clicking through eight different URLs to check on things.

</div>
</div>

---

## Deployment

```bash
# deploy.sh
#!/bin/bash
set -e

NAMESPACE="media"
KUBECONFIG="/path/to/kubeconfig"

# Foundation
kubectl apply -f foundation/namespace.yaml
kubectl apply -f storage/

# Storage
kubectl apply -f storage/media-pvc.yaml

# Apps (order matters)
for app in plex prowlarr sonarr radarr sabnzbd overseerr bazarr tautulli; do
  helm upgrade --install $app bjw-s/app-template \
    -n $NAMESPACE -f apps/$app/values.yaml
done

# Monitoring
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n $NAMESPACE -f apps/monitoring/values.yaml

# Dashboard
kubectl apply -f dashboard/homepage.yaml

echo "✓ Media stack deployed"
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Deploy Prowlarr first. Add your indexers once there, and it pushes them to Sonarr/Radarr via API. Do it in any other order and you'll be manually configuring the same indexers three times.

</div>
</div>

---

## Troubleshooting

> *"The best time to test your backups is before you need them. The second best time is now."* - Disaster Recovery 101

### Sonarr/Radarr Slow

**Symptom:** 30+ second page loads. Console shows SQLite locking warnings.

**Root cause:** SQLite over NFS. This is a known bad combination. The apps do thousands of small random reads. NFS adds 5-10ms to each one.

**Fix:** Move config to local-path storage

```bash
# 1. Backup
kubectl exec -n media deploy/sonarr -- tar czf /tmp/backup.tar.gz -C /config .
kubectl cp media/sonarr-xxx:/tmp/backup.tar.gz /tmp/

# 2. Delete and recreate with local-path
helm uninstall sonarr -n media
kubectl delete pvc sonarr -n media

# Edit values.yaml: storageClass: local-path
helm install sonarr bjw-s/app-template -n media -f apps/sonarr/values.yaml

# 3. Restore
kubectl cp /tmp/backup.tar.gz media/sonarr-xxx:/tmp/
kubectl exec -n media deploy/sonarr -- tar xzf /tmp/backup.tar.gz -C /config
kubectl rollout restart -n media deploy/sonarr
```

### Prometheus Permission Errors

**Symptom:** `open /prometheus/queries.active: permission denied`

**Root cause:** Talos Linux runs a tight security model. Prometheus wants to run as a specific UID that doesn't have permissions on local-path volumes.

**Fix:** Use NFS storage instead of local-path

```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-appdata  # Not local-path
```

Yes, this is backwards from what I told you about databases. Prometheus is the exception. Its workload pattern is fine on NFS, and NFS has looser permissions. Annoying, but it works.

### Plex Not Updating Metadata

**Fix:** Trigger refresh via API

```bash
# Get token
PLEX_TOKEN=$(kubectl exec -n media deploy/plex -- \
  cat "/config/Library/Application Support/Plex Media Server/Preferences.xml" \
  | grep -oP 'PlexOnlineToken="\K[^"]+')

# Refresh library
curl -X PUT "http://<PLEX_IP>:32400/library/sections/1/refresh?X-Plex-Token=$PLEX_TOKEN"
```

### Check Alert Status

```bash
# View active alerts
kubectl port-forward -n media svc/kube-prometheus-stack-alertmanager 9093:9093
# Visit http://localhost:9093

# Check Prometheus rules
kubectl port-forward -n media svc/kube-prometheus-stack-prometheus 9090:9090
# Visit http://localhost:9090/alerts
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Before you close that SSH session, load each app in a browser and make sure it works. I've deployed, disconnected, and then realized I fat-fingered an environment variable. Remote troubleshooting takes 3x longer than fixing it while you're still connected.

</div>
</div>

---

## What Broke (And How I Fixed It)

### SQLite + NFS = Pain

Symptom: 30-45 second page loads  
Root cause: SQLite over NFS  
Solution: Moved databases to local SSDs

This isn't a performance tweak - it's binary. SQLite on NFS is unusable. SQLite on local storage is instant. No middle ground.

### NFS Cache Thrashing

Symptom: High NFS I/O even when apps idle  
Root cause: Apps re-downloading cached images every page load  
Solution: emptyDir volumes for cache directories

Reduced NFS traffic by 60% with 3 lines of YAML. Cache directories should never be on network storage.

### Plex RAM Consumption

Symptom: Node becomes unresponsive under load  
Root cause: Plex consuming all available memory during transcoding  
Solution: Resource limits based on profiled usage

Don't guess. Profile for a week, set limits 20% above peaks. Pod OOMs are recoverable. Node OOMs aren't.

### Restart Loops During Updates

Symptom: Apps crash-loop during version updates  
Root cause: HTTP probes checking before app ready  
Solution: TCP probes with high failureThreshold

These apps take 30-60 seconds to start. HTTP probes fail early, Kubernetes kills the pod, infinite loop. TCP probes just check if the port opens - much more reliable.

---

## What's Next

> *"It works on my cluster."* - Now you can say this unironically

You now have a media stack that you can destroy and rebuild from Git in under 20 minutes. I've done it three times. Twice for hardware upgrades, once because I wanted to test Talos Linux.

That's the point. Infrastructure as code isn't about being clever. It's about having a bad day and knowing you can recover.

What I haven't covered:
- **Backups** - Config is in Git. Media files are replaceable (you have the NZBs, right?). But back up your Plex watch history if you care about it.
- **External access** - Tailscale is the easy answer. Don't expose Plex directly to the internet without Cloudflare in front.
- **GPU transcoding** - Plex with Intel Quick Sync is worth the effort if you have multiple remote users.
- **Scaling** - You don't need it. These apps are single-instance by design. Horizontal scaling doesn't help here.

---

## References

- [Source code for this post](https://github.com/YOUR-USERNAME/k8s-media-stack)
- [bjw-s app-template chart](https://bjw-s-labs.github.io/helm-charts/docs/app-template/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Talos Linux on Proxmox](/2026-02-08-k8s-talos-proxmox-deploy/)
