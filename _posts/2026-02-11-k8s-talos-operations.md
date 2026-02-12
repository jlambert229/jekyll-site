---
title: "Talos Day-2 Operations: Managing Your Cluster"
subtitle: "Everything after the initial deploy. talosctl essentials, etcd management, node replacement, config patches, and troubleshooting a Talos Kubernetes cluster."
date: 2026-02-11
tags:
  - kubernetes
  - talos
  - operations
  - homelab
  - etcd
  - troubleshooting
categories:
  - Kubernetes Homelab
readtime: true
---
Everything after `terraform apply`. talosctl essentials, etcd management, node replacement, config patches, and troubleshooting a Talos cluster.
<!--more-->

> *"The cluster's been running fine for two months. What could go wrong?"* — Narrator: several things

Talos has no SSH. No shell. No `apt`. The entire operating system is managed through the Talos API via `talosctl`. This feels limiting until you realize it means every operation is auditable, scriptable, and reproducible. The cluster's state is never a mystery hidden behind scattered log files on scattered nodes.

---

## The Operator's Toolkit

```bash
export TALOSCONFIG=~/Repos/k8s-deploy/generated/talosconfig
export KUBECONFIG=~/Repos/k8s-deploy/generated/kubeconfig
```

Two files. Two keys to the cluster. Guard them. Back them up. Don't commit them to Git.

---

## talosctl Essentials

### Dashboard

The single most useful command for day-to-day operations:

```bash
talosctl dashboard --nodes 192.168.2.70,192.168.2.80,192.168.2.81
```

A real-time TUI showing CPU, memory, disk, network, and service status across all nodes. One screen replaces five terminal windows of `stats`, `services`, `logs`, `disks`, and `health`. I keep this in a tmux pane whenever I'm working on the cluster. A service going red, memory climbing, a node disappearing — you see it immediately.

### Health Check

```bash
# Quick cluster health
talosctl health --nodes 192.168.2.70,192.168.2.80,192.168.2.81

# With timeout (useful in scripts)
talosctl health --wait-timeout 5m --nodes 192.168.2.70
```

This checks etcd, kubelet, API server, and node connectivity. Run it before and after every maintenance operation. If it fails, stop and investigate — don't push forward hoping the next step will fix it.

### Service Status

```bash
# All services on a node
talosctl services --nodes 192.168.2.70

# Specific service
talosctl services etcd --nodes 192.168.2.70
```

Services include: `apid`, `containerd`, `cri`, `etcd`, `kubelet`, `machined`, `trustd`, `udevd`. If any show as `Unknown` or `Maintenance`, something needs attention.

### Logs

One command, one service, one node. No log directories to traverse, no file paths to remember.

```bash
# Service logs
talosctl logs kubelet --nodes 192.168.2.80
talosctl logs etcd --nodes 192.168.2.70

# Follow mode
talosctl logs kubelet --nodes 192.168.2.80 --follow

# Kernel logs (dmesg)
talosctl dmesg --nodes 192.168.2.80
talosctl dmesg --nodes 192.168.2.80 --follow
```

`talosctl dmesg` is how you debug hardware issues, boot failures, and driver problems. It replaces SSH-ing in and reading `/var/log/` — which doesn't exist on Talos.

### Node Information

```bash
# OS and hardware info
talosctl get members --nodes 192.168.2.70

# Detailed machine config (what's actually applied)
talosctl get machineconfig --nodes 192.168.2.70

# Disk layout
talosctl disks --nodes 192.168.2.80

# Network interfaces
talosctl get addresses --nodes 192.168.2.80
talosctl get routes --nodes 192.168.2.80

# Installed extensions
talosctl get extensions --nodes 192.168.2.80

# System resource usage
talosctl stats --nodes 192.168.2.70
```

---

## etcd Management

etcd is the brain of your cluster. Kubernetes stores all state — Deployments, Services, Secrets, ConfigMaps — in etcd. If etcd is unhealthy, your cluster is unhealthy.

### Health and Members

```bash
# etcd cluster status
talosctl etcd status --nodes 192.168.2.70

# Member list
talosctl etcd members --nodes 192.168.2.70

# etcd alarms (check for NOSPACE, CORRUPT)
talosctl etcd alarm list --nodes 192.168.2.70
```

For a single control plane homelab, you'll see one member. For HA (3 CPs), you'll see three. All should show `started: true`.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

A single control plane node means a single etcd member. If that node dies, etcd is gone. For a homelab that's acceptable — you rebuild from Terraform and restore from [Velero](/post/k8s-velero-backups/). For anything you can't afford to lose, run 3 control plane nodes.

</div>
</div>

### etcd Snapshots

This is your lowest-level backup. Even if Velero fails, an etcd snapshot can reconstruct the entire cluster state.

```bash
# Take a snapshot
talosctl etcd snapshot /tmp/etcd-snapshot.db --nodes 192.168.2.70

# The file is downloaded to your local machine
ls -lh /tmp/etcd-snapshot.db
```

I take snapshots before upgrades, before etcd maintenance, and weekly via cron. Store them outside the cluster — your NAS, S3, wherever.

```bash
#!/bin/bash
SNAPSHOT_DIR="/mnt/nas/backups/etcd"
DATE=$(date +%Y%m%d-%H%M)
talosctl etcd snapshot "${SNAPSHOT_DIR}/etcd-${DATE}.db" --nodes 192.168.2.70
fd -t f --changed-before 30d . "$SNAPSHOT_DIR" -x rm {}
```

### etcd Recovery

If etcd gets corrupted or a member needs to be rebuilt:

```bash
# Remove a stale member
talosctl etcd members --nodes 192.168.2.70
talosctl etcd remove-member <member-id> --nodes 192.168.2.70
```

For full etcd recovery from a snapshot on a fresh cluster:

```bash
talosctl bootstrap --recover-from=/path/to/etcd-snapshot.db \
    --nodes 192.168.2.70
```

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

etcd recovery replays the snapshot into a fresh instance. Kubernetes picks up where the snapshot left off. PVCs will reference volumes that may or may not still exist depending on your storage backend. This is why you need both etcd snapshots AND [Velero backups](/post/k8s-velero-backups/).

</div>
</div>

---

## Node Replacement

Nodes fail. Power supplies die, disks corrupt, VMs get accidentally deleted. With Talos + Terraform, replacing a node is mechanical — not heroic.

### Replace a Worker

**Scenario:** Worker `talos-w-2` (192.168.2.81) has a bad disk.

```bash
# 1. Drain workloads off the node
kubectl drain talos-w-2 --ignore-daemonsets --delete-emptydir-data

# 2. Verify pods migrated
kubectl get pods -A -o wide | rg talos-w-2

# 3. Remove the node from Kubernetes
kubectl delete node talos-w-2

# 4. Destroy and recreate the VM with Terraform
cd ~/Repos/k8s-deploy
terraform taint 'module.worker["talos-w-2"]'
terraform apply
```

Terraform destroys the old VM, creates a new one, applies the Talos machine config, and the node auto-joins. Five minutes, start to finish.

```bash
# 5. Verify
kubectl get nodes
talosctl health --nodes 192.168.2.81 --wait-timeout 5m
```

### Replace the Control Plane

Replacing a single CP is straightforward: snapshot etcd, back up with Velero, taint-and-apply.

```bash
# 1. Take an etcd snapshot FIRST
talosctl etcd snapshot /tmp/etcd-pre-replace.db --nodes 192.168.2.70

# 2. Take a Velero backup
velero backup create pre-cp-replace-$(date +%Y%m%d) \
    --exclude-namespaces kube-system,kube-public,kube-node-lease \
    --default-volumes-to-fs-backup --wait

# 3. Destroy and recreate
cd ~/Repos/k8s-deploy
terraform taint 'module.controlplane["talos-cp-1"]'
terraform apply
```

If the new CP bootstraps successfully, you're done. If not, you have the etcd snapshot and Velero backup to recover from.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ HA is better</div>
<div class="admonition-content" markdown="1">

With 3 control plane nodes, replacing one is safe — the other 2 maintain etcd quorum. With a single CP, you're doing surgery with no safety net. Consider `controlplane_count = 3` in your Terraform config.

</div>
</div>

---

## Config Patches

Talos machine configs are applied at deploy time, but you can patch them on running nodes. The rule is: Terraform is the source of truth. The practical exception: sometimes you need to test a patch live before codifying it.

### View Current Config

```bash
# Full machine config
talosctl get machineconfig -o yaml --nodes 192.168.2.80

# Specific section
talosctl get machineconfig -o yaml --nodes 192.168.2.80 | yq '.spec.machine.network'
```

### Apply a Patch

```bash
# Increase inotify watchers
talosctl patch machineconfig --nodes 192.168.2.80 --patch '[
  {
    "op": "add",
    "path": "/machine/sysctls",
    "value": {
      "fs.inotify.max_user_watches": "524288",
      "fs.inotify.max_user_instances": "8192"
    }
  }
]'
```

Some patches require a reboot. Talos will tell you:

```
Applied configuration without a reboot
```

or:

```
Applied configuration, node will reboot
```

### Common Patches

**Increase inotify limits** (apps that watch many files):

```yaml
machine:
  sysctls:
    fs.inotify.max_user_watches: "524288"
    fs.inotify.max_user_instances: "8192"
```

**Add NTP servers:**

```yaml
machine:
  time:
    servers:
      - time.cloudflare.com
      - pool.ntp.org
```

**Enable host DNS forwarding** (CoreDNS resolves local names):

```yaml
machine:
  features:
    hostDNS:
      enabled: true
      forwardKubeDNSToHost: true
```

**Add a static route:**

```yaml
machine:
  network:
    routes:
      - network: 10.10.0.0/24
        gateway: 192.168.2.1
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Once you've confirmed a patch works, add it to `config_patches` in your Terraform config. Anything not in Terraform eventually gets overwritten by the next `terraform apply`.

</div>
</div>

---

## Certificate Management

Talos manages its own PKI for the Talos API (port 50000) and generates Kubernetes PKI during bootstrap.

### Check Certificate Expiry

```bash
talosctl get certificates --nodes 192.168.2.70
kubectl get csr
```

### Certificate Rotation

Talos handles rotation automatically during upgrades. To force it:

```bash
talosctl rotate-ca --nodes 192.168.2.70,192.168.2.80,192.168.2.81
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

`rotate-ca` is disruptive. All nodes get new certificates, and your existing `talosconfig` becomes invalid until updated. Only do this if certificates are actually expiring or compromised.

</div>
</div>

### kubeconfig Renewal

The kubeconfig has a default expiry (usually 1 year). To regenerate:

```bash
# From Terraform
cd ~/Repos/k8s-deploy
terraform apply

# Or directly
talosctl kubeconfig --nodes 192.168.2.70 > generated/kubeconfig
```

---

## Reset and Wipe

### Reset a Worker

Sometimes a clean slate is the practical choice.

```bash
# Drain first
kubectl drain talos-w-2 --ignore-daemonsets --delete-emptydir-data
kubectl delete node talos-w-2

# Reset (wipes state, keeps Talos installed)
talosctl reset --nodes 192.168.2.81 --graceful

# Node reboots into maintenance mode — re-apply config
talosctl apply-config --nodes 192.168.2.81 \
    --file machine-config-worker.yaml
```

### Full Wipe

For complete decommission:

```bash
talosctl reset --nodes 192.168.2.81 --graceful --wipe-mode all
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

`--wipe-mode all` is destructive and irreversible. Data partition, state partition, ephemeral partition — all destroyed. Drain the node and back up first.

</div>
</div>

---

## Troubleshooting

### Node Won't Join After Rebuild

**Symptoms:** New node created by Terraform, `kubectl get nodes` doesn't show it.

```bash
talosctl services --nodes 192.168.2.81
talosctl logs kubelet --nodes 192.168.2.81
talosctl dmesg --nodes 192.168.2.81 | rg -i 'error|fail|refused'
```

| Cause | Fix |
|-------|-----|
| Old node still registered | `kubectl delete node talos-w-2` then rebuild |
| etcd member stale | `talosctl etcd remove-member <id>` |
| Wrong cluster endpoint | Check `cluster_endpoint` in Terraform vars |
| Firewall blocking 6443 | Check network between nodes |

### etcd Unhealthy

**Symptoms:** `talosctl health` fails at etcd. API server intermittently unavailable.

```bash
talosctl etcd status --nodes 192.168.2.70
talosctl etcd alarm list --nodes 192.168.2.70
talosctl disks --nodes 192.168.2.70
```

| Cause | Fix |
|-------|-----|
| Disk full | etcd needs free space. Check `talosctl disks` |
| NOSPACE alarm | `talosctl etcd alarm disarm --nodes <CP>` after freeing space |
| Stale member | Remove with `talosctl etcd remove-member` |
| Network partition | Check connectivity between CP nodes |

### Node Shows NotReady

```bash
kubectl describe node talos-w-2 | rg -A5 'Conditions'
talosctl services kubelet --nodes 192.168.2.81
talosctl logs kubelet --nodes 192.168.2.81 | tail -50
```

| Cause | Fix |
|-------|-----|
| kubelet stopped | Check `talosctl services kubelet` |
| CNI not ready | Check flannel/cilium pods on that node |
| Node rebooting | Wait. Watch with `talosctl dashboard` |
| Resource exhaustion | `talosctl stats`, check memory/disk |

### Can't Reach Talos API (Port 50000)

```bash
nc -zv 192.168.2.70 50000
```

If the node is in maintenance mode (visible in Proxmox console), it needs a machine config applied before it starts normal services.

### Pods Stuck Terminating After Reboot

```bash
kubectl get pods -A -o wide | rg talos-w-2 | rg Terminating
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

The node went down before kubelet could gracefully terminate pods. Force deletion cleans up the stale records.

---

## Operational Runbook

| Task | Command |
|------|---------|
| Cluster health | `talosctl health --nodes <all-ips>` |
| Real-time dashboard | `talosctl dashboard --nodes <all-ips>` |
| etcd snapshot | `talosctl etcd snapshot /path/to/file --nodes <CP>` |
| etcd members | `talosctl etcd members --nodes <CP>` |
| Node logs | `talosctl logs <service> --nodes <IP>` |
| Kernel logs | `talosctl dmesg --nodes <IP>` |
| Service status | `talosctl services --nodes <IP>` |
| Disk info | `talosctl disks --nodes <IP>` |
| Extensions | `talosctl get extensions --nodes <IP>` |
| Apply config patch | `talosctl patch machineconfig --nodes <IP> --patch '...'` |
| Reset node | `talosctl reset --nodes <IP> --graceful` |
| Drain node | `kubectl drain <name> --ignore-daemonsets --delete-emptydir-data` |
| Version info | `talosctl version --nodes <IP>` |

---

## What I Learned

### 1. The Dashboard Changes Everything

Before `talosctl dashboard`, I ran five commands in five terminals to check cluster health. The dashboard shows all of it on one screen, across all nodes, updating in real time. When I upgrade a node, I watch the services go red, the node disappear, then come back and go green one by one. It turned blind operations into visible ones.

### 2. etcd Snapshots Are Not Optional

I treated etcd snapshots as "nice to have" until a failed upgrade left my single-CP cluster with a corrupted etcd. No snapshot. Rebuilt from Terraform (fast) and restored from Velero (slow — 30 minutes). If I'd had an etcd snapshot, recovery would have been 2 minutes.

The snapshots are small (a few MB for a homelab cluster) and the insurance is priceless.

### 3. Terraform Taint Is Your Friend

First time I needed to replace a node, I manually deleted the VM in Proxmox, then ran `terraform apply`. State said the VM existed, Proxmox said it didn't. Had to `terraform state rm`, then re-import. Messy.

`terraform taint` marks a resource for recreation. Terraform handles the destroy-and-create cleanly. One command to mark, one to apply.

### 4. Config Patches vs Terraform Patches

I patched a running node with `talosctl patch machineconfig` to increase inotify watchers. Two weeks later, `terraform apply` for an unrelated change overwrote my patch. The apps that needed high inotify limits started failing.

Runtime patches are for testing. Terraform is the source of truth. If it's not in Terraform, it eventually disappears.

### 5. No SSH Is a Feature

The first week with Talos, I kept wanting to SSH in. Every time, I found the `talosctl` equivalent. And every time, it was better — it worked across all nodes identically, it was scriptable, and it left no manual changes behind.

SSH encourages poking around: `cd /var/log`, `cat this`, `grep that`. `talosctl` forces structured debugging: check services, read logs, examine config. One tool, one way. After a month, I wouldn't go back.

---

## What's Next

- [Upgrade Talos and Kubernetes](/post/k8s-talos-upgrades/) — Rolling upgrade procedures
- [Deploy the cluster](/post/k8s-talos-proxmox-deploy/) — Initial setup with Terraform
- [Velero Backups](/post/k8s-velero-backups/) — Cluster-level disaster recovery
- [Cluster Autoscaling](/post/k8s-talos-autoscaling/) — HPA, VPA, and node autoscaling

---

## References

- [Talos CLI Reference](https://www.talos.dev/latest/reference/cli/)
- [Talos Machine Configuration](https://www.talos.dev/latest/reference/configuration/)
- [Talos Troubleshooting Guide](https://www.talos.dev/latest/introduction/troubleshooting/)
- [etcd Operations with Talos](https://www.talos.dev/latest/advanced/etcd-maintenance/)
- [talosctl Cheat Sheet](https://www.talos.dev/latest/talos-guides/configuration/)
