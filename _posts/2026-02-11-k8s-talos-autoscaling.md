---
title: "Autoscaling a Talos Kubernetes Cluster"
subtitle: "Pod and node autoscaling for a homelab Talos cluster. HPA for horizontal scaling, VPA for right-sizing, and a custom Terraform-based node autoscaler for Proxmox."
date: 2026-02-11
tags:
  - kubernetes
  - talos
  - autoscaling
  - homelab
  - proxmox
  - terraform
categories:
  - Kubernetes Homelab
readtime: true
---
Pod and node autoscaling for a homelab Talos cluster. HPA for horizontal scaling, VPA for right-sizing, and a custom Terraform-based node autoscaler for Proxmox.
<!--more-->

> *"My Plex transcode pod needs 4 cores right now, but 0.1 cores at 3am. I'm not manually scaling this."*

---

## The Problem

Static resource allocation wastes capacity. You size workloads for peak load, then spend 90% of the day running at 20% utilization. On a homelab with limited RAM and CPU, that overhead means fewer apps or bigger hardware.

I ran everything with fixed requests and limits for months. Sonarr got 512Mi of RAM — it used 180Mi. Plex got 2 CPU cores — it used 0.3 cores unless someone was transcoding. Meanwhile, qBittorrent was OOMKilled weekly because its 512Mi limit was too low during batch imports.

The fix isn't guessing better numbers. It's letting the cluster measure actual usage and adjust.

---

## Architecture

Three layers of autoscaling, each solving a different problem:

```
┌────────────────────────────────────────────────────────┐
│  Layer 3: Node Autoscaler (custom)                     │
│    Monitors: kubectl top nodes                         │
│    Adjusts: worker_count in terraform.tfvars           │
│    Result: More/fewer Proxmox VMs                      │
├────────────────────────────────────────────────────────┤
│  Layer 2: HPA (Horizontal Pod Autoscaler)              │
│    Monitors: Pod CPU/memory metrics                    │
│    Adjusts: Replica count                              │
│    Result: More/fewer pods                             │
├────────────────────────────────────────────────────────┤
│  Layer 1: VPA (Vertical Pod Autoscaler)                │
│    Monitors: Pod resource usage over time              │
│    Adjusts: Resource requests/limits                   │
│    Result: Right-sized pods                            │
├────────────────────────────────────────────────────────┤
│  Foundation: Metrics Server                            │
│    Provides: CPU/memory metrics via Metrics API        │
│    Required by: HPA, VPA, kubectl top                  │
└────────────────────────────────────────────────────────┘
```

| Layer | Scales what | Trigger | Speed |
|-------|-------------|---------|-------|
| **VPA** | Pod resources (CPU/RAM) | Usage drift from requests | Minutes (recreates pod) |
| **HPA** | Pod replicas | CPU/memory threshold | Seconds |
| **Node autoscaler** | Worker VMs | Node-level CPU threshold | Minutes (Terraform apply) |

Full source: [k8s-deploy/addons](https://github.com/YOUR-USERNAME/k8s-deploy/tree/main/addons)

---

## Metrics Server

Everything depends on metrics. No metrics, no autoscaling.

```bash
kubectl apply -f addons/metrics-server/metrics-server.yaml

# Wait for it
kubectl wait --for=condition=ready pod \
    -l k8s-app=metrics-server -n kube-system --timeout=120s
```

Verify:

```bash
kubectl top nodes
# NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# talos-cp-1    250m         12%    1200Mi          30%
# talos-w-1     800m         20%    3200Mi          20%
# talos-w-2     600m         15%    2800Mi          17%

kubectl top pods -n media
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

If `kubectl top` shows `error: Metrics API not available`, the metrics server isn't ready yet. Check its logs: `kubectl logs -n kube-system -l k8s-app=metrics-server`. Common issue: metrics server can't verify kubelet certificates — Talos uses self-signed certs, so the metrics server deployment needs `--kubelet-insecure-tls`.

</div>
</div>

---

## Vertical Pod Autoscaler (VPA)

VPA watches actual pod resource consumption over time and recommends (or applies) better requests and limits. It's the "stop guessing" autoscaler.

### Deploy

```bash
kubectl apply -f addons/vpa/vpa.yaml
kubectl get pods -n vpa-system
```

### VPA Modes

| Mode | Behavior | When to use |
|------|----------|-------------|
| `Off` | Recommends only, doesn't change anything | Start here. Observe for a week. |
| `Initial` | Sets requests on pod creation, no restarts | Stateful apps (Plex, Sonarr) |
| `Auto` | Evicts and recreates pods with new requests | Stateless apps (frontends, APIs) |

### Example: VPA for Sonarr

Start in `Off` mode to see what VPA recommends without disrupting anything:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: sonarr
  namespace: media
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sonarr
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 50m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 4Gi
```

After a week, check what VPA thinks:

```bash
kubectl describe vpa sonarr -n media
```

You'll see:

```
Recommendation:
  Container Recommendations:
    Container Name: app
    Lower Bound:    Cpu: 80m,  Memory: 160Mi
    Target:         Cpu: 150m, Memory: 280Mi
    Upper Bound:    Cpu: 400m, Memory: 600Mi
    Uncapped Target: Cpu: 150m, Memory: 280Mi
```

This tells you Sonarr actually needs ~150m CPU and ~280Mi RAM — half of what I'd guessed. Apply those numbers to your Helm values or switch the VPA to `Initial` mode to let it set requests on new pods.

### VPA for the Whole Media Stack

```yaml
# Apply VPA to every media app
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: plex
  namespace: media
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: plex
  updatePolicy:
    updateMode: "Initial"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 256Mi
      maxAllowed:
        cpu: 4000m
        memory: 8Gi
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

Don't use `Auto` mode on Plex or other media apps that hold long-running connections. VPA in `Auto` mode evicts pods to apply new resources — killing active transcodes or downloads. Use `Initial` for media workloads: it only sets resources when the pod is first created.

</div>
</div>

---

## Horizontal Pod Autoscaler (HPA)

HPA adds or removes pod replicas based on metrics. Most useful for stateless workloads that can run multiple instances.

### When HPA Makes Sense

| App | HPA useful? | Why |
|-----|-------------|-----|
| Overseerr | Yes | Stateless frontend, handles more users with more replicas |
| API services | Yes | Horizontal scaling is natural |
| Plex | No | Single instance owns the media library |
| Sonarr/Radarr | No | SQLite DB, not designed for multi-instance |
| qBittorrent | No | Port mappings, single-instance tracker connections |

### Example: HPA for Overseerr

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: overseerr
  namespace: media
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: overseerr
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
    scaleUp:
      policies:
      - type: Pods
        value: 1
        periodSeconds: 30
```

The `behavior` section prevents flapping — scale down waits 5 minutes before removing a replica, and only removes one at a time. Scale up is faster: one replica every 30 seconds.

### Monitor HPA

```bash
kubectl get hpa -n media
# NAME        REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
# overseerr   Deployment/overseerr  15%/70%   1         3         1

kubectl describe hpa overseerr -n media
```

### HPA Prerequisites

HPA only works if pods have resource requests set. Without requests, there's nothing to calculate a percentage of.

```yaml
# This MUST exist for HPA to work
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

If HPA shows `<unknown>/70%`, it means requests aren't set.

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

Don't use HPA and VPA on the same metric for the same pod. If both try to adjust CPU, they'll fight. The pattern is: VPA manages `requests` (right-sizing), HPA manages replica count (horizontal scaling). VPA sets the right size per pod; HPA decides how many pods.

</div>
</div>

---

## Node Autoscaler

Cloud Kubernetes (EKS, GKE, AKS) has built-in node autoscaling. Bare-metal Proxmox doesn't. So I built a simple one.

### How It Works

A Python service monitors cluster-wide CPU usage. When average CPU exceeds a threshold, it bumps `worker_count` in `terraform.tfvars` and runs `terraform apply`. Talos auto-joins the new node. When CPU drops, it reduces `worker_count` and Terraform destroys the extra VM.

```
Monitor (every 60s)
  ↓
kubectl top nodes → average CPU%
  ↓
CPU > 80%? → Scale up (worker_count + 1, terraform apply)
CPU < 30%? → Scale down (worker_count - 1, terraform apply)
  ↓
Cooldown (5 minutes between operations)
```

### Configuration

```bash
# /home/owner/Repos/k8s-deploy/addons/node-autoscaler/autoscaler.env
SCALE_UP_THRESHOLD=80
SCALE_DOWN_THRESHOLD=30
MIN_WORKERS=2
MAX_WORKERS=6
CHECK_INTERVAL=60
COOLDOWN_SECONDS=300
```

| Setting | Value | Why |
|---------|-------|-----|
| `MIN_WORKERS=2` | Floor | Workloads need at least 2 nodes for scheduling redundancy |
| `MAX_WORKERS=6` | Ceiling | Proxmox host has 64GB RAM; 6 workers × 16GB = 96GB, so 6 is the limit |
| `SCALE_UP_THRESHOLD=80` | Trigger | 80% average CPU means pods are contending |
| `SCALE_DOWN_THRESHOLD=30` | Trigger | 30% means wasted capacity |
| `COOLDOWN_SECONDS=300` | Guard | Prevents rapid oscillation |

### Deploy as Systemd Service

```bash
cd ~/Repos/k8s-deploy/addons/node-autoscaler

# Install dependencies
python3 -m venv venv
venv/bin/pip install -r requirements.txt

# Install service
sudo cp autoscaler.service /etc/systemd/system/k8s-node-autoscaler.service
sudo systemctl daemon-reload
sudo systemctl enable --now k8s-node-autoscaler
```

Monitor it:

```bash
sudo systemctl status k8s-node-autoscaler
sudo journalctl -u k8s-node-autoscaler -f
```

### Test Scale-Up

Simulate load to verify the autoscaler responds:

```bash
# Deploy a CPU stress test
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cpu-stress
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "4", "--timeout", "600s"]
    resources:
      requests:
        cpu: "2000m"
EOF

# Watch autoscaler respond
sudo journalctl -u k8s-node-autoscaler -f
# After ~60s: "Average CPU: 85%. Scaling up from 2 to 3 workers"
# Terraform runs, new VM appears, Talos joins it

# Clean up
kubectl delete pod cpu-stress
```

### Test Scale-Down

After the stress test, wait for cooldown (5 minutes), then watch:

```bash
# Autoscaler should detect low CPU and scale down
sudo journalctl -u k8s-node-autoscaler -f
# "Average CPU: 22%. Scaling down from 3 to 2 workers"
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

The node autoscaler runs `terraform apply -auto-approve`. This is intentional for automation, but it means Terraform state must be consistent. If someone is manually editing `terraform.tfvars` while the autoscaler runs, you'll get conflicts. Don't manually edit `worker_count` while the autoscaler is active.

</div>
</div>

---

## Putting It All Together

The recommended setup for a homelab media cluster:

| Component | Scope | Mode |
|-----------|-------|------|
| **Metrics Server** | Cluster-wide | Always on |
| **VPA** | All media apps | `Off` for first week, then `Initial` |
| **HPA** | Stateless apps only (Overseerr, Homepage) | 1-3 replicas |
| **Node autoscaler** | Whole cluster | 2-6 workers, 80/30 thresholds |

### Deployment Order

1. **Metrics Server** — Everything depends on this
2. **VPA** — Start in `Off` mode, observe recommendations
3. **HPA** — Only on apps that support multi-instance
4. **Node autoscaler** — Optional, useful if load varies significantly

```bash
# 1. Metrics
kubectl apply -f addons/metrics-server/metrics-server.yaml
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=120s

# 2. VPA
kubectl apply -f addons/vpa/vpa.yaml

# 3. Verify
kubectl top nodes
kubectl top pods -A
kubectl get vpa -A
```

---

## Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `kubectl top` shows `<unknown>` | Metrics server not running | Check `kubectl get pods -n kube-system -l k8s-app=metrics-server` |
| HPA shows `<unknown>/70%` | Pod has no resource requests | Add `resources.requests` to the Deployment |
| VPA recommendations seem wrong | Not enough data | Wait a week. VPA needs usage history to make good recommendations |
| VPA + HPA fighting | Both managing CPU | Use VPA for requests, HPA for replicas. Don't overlap metrics |
| Node autoscaler flapping | Thresholds too close | Widen the gap: 80% up / 30% down, not 70% up / 50% down |
| Scale-down kills active workloads | Pods not draining | Node autoscaler should drain before removing. Check logs |
| Terraform conflict | Manual tfvars edit during autoscale | Stop autoscaler before manual changes: `sudo systemctl stop k8s-node-autoscaler` |

---

## What I Learned

### 1. VPA in Off Mode Is Worth More Than You'd Think

I deployed VPA in `Off` mode "just to see what it says." A week later, the recommendations changed how I thought about resource allocation. Plex: I gave it 2 CPU / 4 Gi. VPA said it actually used 0.4 CPU / 1.2 Gi — except during transcodes, when it spiked to 3 CPU. Sonarr: I gave it 512Mi. VPA said 180Mi. Radarr: I gave it 512Mi. VPA said 200Mi.

I was over-provisioning everything by 2-3x. On a 2-worker cluster with 32 GB total RAM, that's the difference between "can't schedule anything else" and "room for 5 more apps." Even if you never enable `Auto` mode, the recommendations alone justify deploying VPA.

### 2. HPA Doesn't Make Sense for Most Homelab Apps

I tried HPA on Sonarr. Replicas scaled to 2 during a mass import. Both replicas wrote to the same SQLite database. Corruption. Lost the database, restored from Velero, spent an hour re-importing custom formats.

Most *arr apps and media tools are fundamentally single-instance. They use local databases, write to shared filesystems with single-writer assumptions, and maintain state in memory. HPA is for stateless services — web frontends, APIs, proxies. For a homelab, that's maybe Overseerr and Homepage. Everything else should stay at 1 replica with VPA for right-sizing.

### 3. Node Autoscaling Has a Startup Tax

A new Proxmox VM takes about 2 minutes to boot, get a Talos config, start kubelet, and register with the API server. Add 1-2 minutes for pods to schedule and start on the new node. Total: 3-4 minutes from scale-up trigger to usable capacity.

For a homelab, this is fine — media workloads aren't latency-critical. But it means the autoscaler is reactive, not predictive. If you know you'll need extra capacity (movie night with the family, batch import), scale up manually before the load hits:

```bash
cd ~/Repos/k8s-deploy
# Edit worker_count = 4 in terraform.tfvars
terraform apply
```

### 4. Cooldowns Prevent Expensive Oscillation

Without the 5-minute cooldown, the autoscaler would: scale up (terraform apply, 3 minutes), measure CPU (now low because new node), scale down (terraform apply, 3 minutes), measure CPU (now high because node removed), scale up again. Each cycle costs 6 minutes of Terraform applies and Proxmox VM operations.

The 300-second cooldown is the minimum that prevents this. I initially set it to 60 seconds and watched it add and remove a node three times in 20 minutes. The Proxmox host was not amused.

---

## What's Next

- [Talos Day-2 Operations](/2026-02-11-k8s-talos-operations/) - Node management, etcd, troubleshooting
- [Upgrade Talos and Kubernetes](/2026-02-11-k8s-talos-upgrades/) - Rolling upgrade procedures
- [Deploy the cluster](/2026-02-08-k8s-talos-proxmox-deploy/) - Initial Terraform setup
- [Velero Backups](/2026-02-08-k8s-velero-backups/) - Disaster recovery

---

## References

- [Kubernetes HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [k8s-deploy Autoscaling Addons](https://github.com/YOUR-USERNAME/k8s-deploy/tree/main/addons)
- [Talos Linux Cluster Autoscaler Discussion](https://github.com/siderolabs/talos/discussions/3939)
