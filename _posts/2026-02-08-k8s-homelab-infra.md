---
title: "Kubernetes Homelab Infrastructure: MetalLB, Traefik, and NFS"
subtitle: "Bare-metal LoadBalancer, ingress controller, and persistent storage for a Talos Kubernetes cluster. The foundation layer before deploying workloads."
date: 2026-02-08
tags:
  - kubernetes
  - homelab
  - traefik
  - metallb
  - storage
  - getting-started
categories:
  - Kubernetes Homelab
readtime: true
---
Common problems and solutions for running K8s on bare metal. LoadBalancer stuck pending? Ingress returning 404? PVCs won't bind? Here's how to fix it.
<!--more-->

I spent an hour convinced my Ingress was wrong. Checked the YAML. Checked the Service. Checked Traefik logs. The real issue: I'd pointed DNS at the *old* Traefik IP from before a reinstall. The cluster was fine. I was debugging the wrong problem. This guide exists so you skip that hour.

Source: [k8s-media-stack](https://github.com/YOUR-USERNAME/k8s-media-stack)

---

## Why doesn't LoadBalancer work?

**Problem:** Service with `type: LoadBalancer` stuck at `<pending>`

**Why:** Cloud providers automatically provision load balancers. Bare metal doesn't. No controller = no IP assignment.

**Solution:** Install MetalLB

```bash
helm repo add metallb https://metallb.github.io/metallb
helm upgrade --install metallb metallb/metallb -n metallb-system --create-namespace --wait
kubectl rollout status -n metallb-system deploy/metallb-controller --timeout=90s
```

Create an IP pool (pick IPs outside your DHCP range):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lan-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.244-192.168.1.254
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lan-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - lan-pool
```

```bash
kubectl apply -f metallb-config.yaml
```

**Test it:**

```bash
kubectl create deployment nginx-test --image=nginx
kubectl expose deployment nginx-test --type=LoadBalancer --port=80
kubectl get svc nginx-test  # Should show EXTERNAL-IP
curl http://<EXTERNAL-IP>
kubectl delete deployment nginx-test && kubectl delete svc nginx-test
```

---

## Why can't I access apps by hostname?

**Problem:** Want to access services via `app.media.lan` instead of IP:port

**Why:** No ingress controller installed. K8s doesn't do hostname routing out of the box.

**Solution:** Install Traefik

```yaml
# traefik-values.yaml
service:
  type: LoadBalancer
  annotations:
    metallb.universe.tf/loadBalancerIPs: "192.168.1.244"  # Pick an IP
ports:
  web: {exposedPort: 80}
  websecure: {exposedPort: 443}
providers:
  kubernetesIngress: {enabled: true, publishedService: {enabled: true}}
```

```bash
helm repo add traefik https://traefik.github.io/charts
helm upgrade --install traefik traefik/traefik -n traefik --create-namespace -f traefik-values.yaml --wait
```

**Check it worked:**

```bash
kubectl get svc -n traefik  # EXTERNAL-IP should show your pinned IP
```

**Create an Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: traefik
  rules:
  - host: myapp.media.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: myapp, port: {number: 80}}
```

**Set DNS:** Point `*.media.lan` → Traefik IP

**Test without DNS:**

```bash
curl -H "Host: myapp.media.lan" http://<TRAEFIK_IP>
```

Works? DNS is wrong. 404? Ingress is wrong.

## Why is my Ingress returning 404?

Check list:
1. Is `ingressClassName: traefik` set?
2. Does DNS point to Traefik's IP?
3. Is the backend service name correct?
4. Is the backend service port correct?

```bash
kubectl get ingress -n <namespace>  # Check status
kubectl get svc -n <namespace>      # Verify service exists
kubectl describe ingress <name> -n <namespace>  # See routing rules
```

---

## Why won't my PVC bind?

**Problem:** PVC stuck in `Pending` status

**Why:** Usually one of:
- No StorageClass configured
- CSI driver not running
- NFS server unreachable

**Solution:** Install NFS CSI driver (works with Talos)

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm upgrade --install csi-driver-nfs csi-driver-nfs/csi-driver-nfs -n kube-system --wait
```

Create StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-appdata
provisioner: nfs.csi.k8s.io
parameters:
  server: "192.168.1.100"
  share: "/volume1/k8s-appdata"
reclaimPolicy: Retain
mountOptions: [nfsvers=3, nolock]
```

```bash
kubectl apply -f nfs-storageclass.yaml
```

**Use it in a PVC:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-config
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-appdata
  resources:
    requests:
      storage: 2Gi
```

## Pod stuck in ContainerCreating?

Check NFS mount errors:

```bash
kubectl describe pod <pod-name>
```

Look for mount errors. Common causes:
- NFS export not configured on NAS
- Firewall blocking NFS (port 2049)
- Wrong server IP or share path
- NFS permissions issue

Fix on NAS side, then delete/recreate the pod.

## Need ReadWriteMany for shared storage?

Create static PV/PVC:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: shared-media
spec:
  capacity: {storage: 10Ti}
  accessModes: [ReadWriteMany]
  csi:
    driver: nfs.csi.k8s.io
    volumeHandle: shared-media
    volumeAttributes: {server: "192.168.1.100", share: "/volume1/media"}
  mountOptions: [nfsvers=3, nolock]
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-media
  namespace: media
spec:
  accessModes: [ReadWriteMany]
  storageClassName: ""
  resources: {requests: {storage: 10Ti}}
  volumeName: shared-media
```

Multiple pods can mount this simultaneously. Useful for media stacks where download client and library manager both need the same files.

---

## Quick deployment order

```bash
# Add repos
helm repo add metallb https://metallb.github.io/metallb
helm repo add traefik https://traefik.github.io/charts
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update

# 1. MetalLB
helm upgrade --install metallb metallb/metallb -n metallb-system --create-namespace --wait
kubectl apply -f metallb-config.yaml

# 2. NFS CSI
helm upgrade --install csi-driver-nfs csi-driver-nfs/csi-driver-nfs -n kube-system --wait
kubectl apply -f nfs-storageclass.yaml

# 3. Traefik
helm upgrade --install traefik traefik/traefik -n traefik --create-namespace -f traefik-values.yaml --wait
```

**Verify:**

```bash
kubectl get pods -n metallb-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=csi-driver-nfs
kubectl get svc -n traefik
```

---

## Quick troubleshooting reference

| Problem | Check | Fix |
|---------|-------|-----|
| LoadBalancer `<pending>` | `kubectl get pods -n metallb-system` | Install MetalLB, create IPAddressPool |
| Traefik wrong IP | `kubectl get ipaddresspools -A` | Pin with `metallb.universe.tf/loadBalancerIPs` |
| PVC `Pending` | `kubectl get pods -n kube-system -l app=csi-driver-nfs` | Install CSI driver, check NFS reachable |
| Pod `ContainerCreating` | `kubectl describe pod <name>` | Check NFS mount error, fix NAS exports |
| Ingress 404 | `kubectl get ingress`, check DNS | Set `ingressClassName: traefik`, fix DNS |

---

## Related

- [Media Stack on Kubernetes](/post/k8s-media-stack/) - Deploy Plex, Sonarr, Radarr on this foundation
- [Deploy cluster with Talos + Terraform](/post/k8s-talos-proxmox-deploy/) - Automated cluster provisioning
- [GitOps with Flux](/post/k8s-flux-gitops/) - Git as source of truth
- [Velero Backups](/post/k8s-velero-backups/) - Cluster-level disaster recovery

[MetalLB](https://metallb.io/) • [Traefik](https://doc.traefik.io/traefik/) • [NFS CSI](https://github.com/kubernetes-csi/csi-driver-nfs) • [Source](https://github.com/YOUR-USERNAME/k8s-media-stack)
