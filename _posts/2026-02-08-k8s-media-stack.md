---
title: "Media Stack on Kubernetes with Helm"
subtitle: "Plex, Sonarr, Radarr, and friends deployed on Kubernetes with the bjw-s app-template Helm chart, shared NFS storage, and Traefik ingress."
date: 2026-02-08
tags:
  - kubernetes
  - homelab
  - helm
  - plex
  - media
  - getting-started
categories:
  - Kubernetes Homelab
readtime: true
---
Quick reference for deploying media apps on K8s using `bjw-s/app-template`.
<!--more-->

*"Stay frosty."* - Aliens. Media stacks have many moving parts. Same Helm chart template, different values. Stay organized.

Prerequisites: [K8s infrastructure](/2026-02-08-k8s-homelab-infra/) (MetalLB, Traefik, NFS CSI)

> *"Same chart, different values. Standardization reduces cognitive load. You learn one pattern, apply it everywhere."* - Helm best practice

Source: [k8s-media-stack](https://github.com/YOUR-USERNAME/k8s-media-stack)

---

## Stack

| App | Port | Ingress |
|-----|------|---------|
| Plex | 32400 | LoadBalancer IP |
| Sonarr | 8989 | sonarr.media.lan |
| Radarr | 7878 | radarr.media.lan |
| Prowlarr | 9696 | prowlarr.media.lan |
| qBittorrent | 8080 | qbit.media.lan |
| Bazarr | 6767 | bazarr.media.lan |
| Overseerr | 5055 | overseerr.media.lan |
| Tautulli | 8181 | tautulli.media.lan |

## Storage

**Shared NFS (ReadWriteMany)** - `/data` mounted in all apps for hardlinks

```
/data/
  media/
    tv/      # Sonarr + Plex
    movies/  # Radarr + Plex
  downloads/
    complete/   # qBittorrent output
    incomplete/ # In progress
```

**Per-app config (ReadWriteOnce)** - Dynamically provisioned via `nfs-appdata` StorageClass

## Helm Pattern

All apps use `bjw-s/app-template`. Same structure, different values:

```yaml
controllers:
  sonarr:
    strategy: Recreate  # SQLite - no concurrent access
    containers:
      app:
        image: {repository: linuxserver/sonarr, tag: latest}
        env: {PUID: "1000", PGID: "1000", TZ: "America/New_York"}

service:
  app: {controller: sonarr, ports: {http: {port: 8989}}}

ingress:
  app:
    className: traefik
    hosts: [{host: sonarr.media.lan, paths: [{path: /, service: {identifier: app, port: http}}]}]

persistence:
  config: {type: persistentVolumeClaim, size: 2Gi, storageClass: nfs-appdata, globalMounts: [{path: /config}]}
  data: {type: persistentVolumeClaim, existingClaim: media-data, globalMounts: [{path: /data}]}
```

**Key points:**
- `strategy: Recreate` - SQLite doesn't do concurrent writes
- `PUID/PGID` - Match NFS ownership
- Two volumes: `/config` (per-app), `/data` (shared)

### Plex Exception

```yaml
service:
  app:
    controller: plex
    type: LoadBalancer
    annotations: {metallb.universe.tf/loadBalancerIPs: "<PLEX_IP>"}
    ports: {http: {port: 32400}}
persistence:
  transcode: {type: emptyDir, globalMounts: [{path: /transcode}]}  # Fast local scratch
```

Get claim token: [plex.tv/claim](https://plex.tv/claim) (4min validity)

## Deploy

```bash
helm repo add bjw-s https://bjw-s-labs.github.io/helm-charts/
kubectl apply -f storage/media-data.yaml

helm upgrade --install sonarr bjw-s/app-template -n media -f apps/sonarr/values.yaml
# Repeat for radarr, prowlarr, qbittorrent, bazarr, overseerr, tautulli
```

**DNS:** Point `*.media.lan` → `<TRAEFIK_IP>`

**App configuration order:**
1. Plex - add libraries
2. qBittorrent - set paths
3. Sonarr/Radarr - add download client
4. Prowlarr - add indexers
5. Bazarr, Overseerr - connect upstream apps

Use K8s DNS for inter-app communication: `<app>.media.svc.cluster.local:<port>`

## Resources

| App | CPU | RAM |
|-----|-----|-----|
| Plex | 500m/2000m | 512Mi/2Gi |
| Sonarr/Radarr | 100m/500m | 256Mi/512Mi |
| Others | 50m/250m | 128Mi/256Mi |
| **Total** | **1050m** | **1.7Gi** |

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Pod stuck `ContainerCreating` | Check NFS mount/CSI driver |
| App won't start | Verify `PUID`/`PGID` match NFS ownership |
| Hardlinks fail | Ensure both paths under same `/data` mount |
| Plex unreachable | Check MetalLB IP assignment |
| Ingress 404 | Verify DNS → Traefik IP |

## Next

[GitOps with Flux](/2026-02-08-k8s-flux-gitops/) - automate deployments from Git

---

## Refs

[Source](https://github.com/YOUR-USERNAME/k8s-media-stack) • [bjw-s chart](https://bjw-s-labs.github.io/helm-charts/docs/app-template/) • [LinuxServer images](https://docs.linuxserver.io/) • [Hardlinks guide](https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/)
