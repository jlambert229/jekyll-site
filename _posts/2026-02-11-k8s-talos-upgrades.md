---
title: "Upgrading a Talos Kubernetes Cluster"
subtitle: "Rolling upgrades for Talos Linux and Kubernetes versions. Upgrade workers first, control plane last, and always have a rollback plan."
date: 2026-02-11
tags:
  - kubernetes
  - talos
  - upgrades
  - homelab
  - operations
categories:
  - Kubernetes Homelab
readtime: true
---
Rolling upgrades for Talos Linux and Kubernetes. Workers first, control plane last. Always have a rollback plan.
<!--more-->

> *"I'll just upgrade everything at once, it's a homelab"* — Past me, before 3 hours of downtime

---

## The Two Things You Upgrade

A Talos cluster has two independent version numbers:

1. **Talos Linux** — The operating system on each node (e.g., v1.9.2 → v1.9.4)
2. **Kubernetes** — The container orchestrator running on top (e.g., v1.32.0 → v1.32.2)

These are upgraded separately, with different commands, and they can drift. The [Talos support matrix](https://www.talos.dev/latest/introduction/support-matrix/) shows which Kubernetes versions each Talos release supports. Check it before every upgrade.

| Upgrade | Command | Restarts node? | Restarts workloads? |
|---------|---------|---------------|---------------------|
| Talos Linux | `talosctl upgrade` | Yes (reboot) | Yes (node drains) |
| Kubernetes | `talosctl upgrade-k8s` | No | No (rolling update of system pods) |

**Order matters:** Upgrade Talos first, then Kubernetes. The new Talos version may ship with updated kubelet defaults that the new Kubernetes version expects.

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

Talos minor releases (v1.9 → v1.10) can change machine config schemas and API behavior. Patch releases (v1.9.2 → v1.9.4) are bug fixes and safe. Read the [release notes](https://github.com/siderolabs/talos/releases) before any minor bump.

</div>
</div>

---

## Pre-Flight Checklist

Before touching anything, verify the cluster is healthy. I upgraded on a broken cluster exactly once. etcd had a stale member from a previous node replacement. The upgrade triggered a leader election that failed because quorum was already fragile. Two hours to recover. Now I check everything first.

```bash
# 1. Cluster health (all nodes must be Ready)
export TALOSCONFIG=generated/talosconfig
talosctl health --nodes <CP_IP>,<W1_IP>,<W2_IP>

# 2. Kubernetes health
export KUBECONFIG=generated/kubeconfig
kubectl get nodes
kubectl get pods -A | rg -v 'Running|Completed'

# 3. etcd health (critical for CP upgrades)
talosctl etcd status --nodes <CP_IP>
talosctl etcd members --nodes <CP_IP>

# 4. Current versions
talosctl version --nodes <CP_IP>
kubectl version --short

# 5. Check for pending changes (don't upgrade during a Flux reconciliation)
flux get kustomizations
flux get helmreleases -A | rg -v 'True'
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Backup first</div>
<div class="admonition-content" markdown="1">

Take a [Velero backup](/2026-02-08-k8s-velero-backups/) before upgrading. If the upgrade goes wrong, you want to restore workloads to a fresh cluster, not debug a half-upgraded one.

```bash
velero backup create pre-upgrade-$(date +%Y%m%d) \
    --exclude-namespaces kube-system,kube-public,kube-node-lease \
    --default-volumes-to-fs-backup --wait
```

</div>
</div>

---

## Upgrading Talos Linux

### Get Your Schematic ID

Every Talos cluster built with Image Factory has a schematic ID that encodes your extensions. You need this for the upgrade image URL.

```bash
# From Terraform output (if you deployed with the k8s-deploy repo)
terraform output -raw talos_schematic_id

# Or from a running node
talosctl get extensions --nodes <NODE_IP>
```

The schematic ID is a hash like `376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba`. It preserves your extension set (qemu-guest-agent, iscsi-tools, etc.) across upgrades.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

If you've changed your extensions since the last deploy, generate a new schematic at [factory.talos.dev](https://factory.talos.dev/) or let Terraform handle it by updating `talos_extensions` in your tfvars and running `terraform apply`.

</div>
</div>

### Upgrade Workers First

Workers first. Always.

Workers are stateless from Kubernetes' perspective. If an upgrade fails on a worker, your control plane is untouched and workloads reschedule to healthy nodes.

```bash
SCHEMATIC_ID=$(terraform output -raw talos_schematic_id)
NEW_VERSION="v1.9.4"

# Upgrade worker 1
talosctl upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_VERSION} \
    --preserve \
    --nodes 192.168.1.80

# Wait for it to come back
talosctl health --nodes 192.168.1.80 --wait-timeout 5m

# Verify version
talosctl version --nodes 192.168.1.80
```

The node reboots during upgrade. Kubernetes drains it first, so pods migrate to other workers. With `--preserve`, the data partition survives — PVCs backed by local storage stay intact.

**Repeat for each worker.** One at a time. Verify. Move on.

```bash
# Worker 2
talosctl upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_VERSION} \
    --preserve \
    --nodes 192.168.1.81

talosctl health --nodes 192.168.1.81 --wait-timeout 5m
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

Don't upgrade all workers simultaneously. If the new version has a bug, you've taken down every worker at once. One node, one verification, one step at a time.

</div>
</div>

### Upgrade Control Plane

After all workers are on the new version and healthy:

```bash
talosctl upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_VERSION} \
    --preserve \
    --nodes 192.168.1.70
```

For a single CP homelab, the API server goes down during the reboot (30–90 seconds). Workloads keep running on workers — they just can't be managed until the API comes back.

```bash
# Wait for CP to return
talosctl health --nodes 192.168.1.70 --wait-timeout 10m

# Verify everything
kubectl get nodes
talosctl version --nodes 192.168.1.70,192.168.1.80,192.168.1.81
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 HA Control Plane</div>
<div class="admonition-content" markdown="1">

With 3 control plane nodes, upgrade one at a time. etcd maintains quorum with 2/3 nodes, so the API stays available throughout. Upgrade order: secondary CPs first, primary last.

```bash
for node in 192.168.1.71 192.168.1.72 192.168.1.70; do
    talosctl upgrade \
        --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_VERSION} \
        --preserve --nodes $node
    talosctl health --nodes $node --wait-timeout 10m
done
```

</div>
</div>

---

## Upgrading Kubernetes

This is the gentler upgrade. No reboots, no node drains. Talos updates the Kubernetes components (API server, controller-manager, scheduler, kube-proxy, kubelet) in a rolling fashion.

```bash
NEW_K8S="1.32.2"

talosctl upgrade-k8s \
    --to ${NEW_K8S} \
    --nodes 192.168.1.70
```

You only run this against a control plane node. Talos handles updating all nodes.

```bash
# Watch the upgrade progress
talosctl dmesg --nodes 192.168.1.70 --follow | rg -i 'upgrade|kubernetes'

# Verify
kubectl version
kubectl get nodes -o wide
```

The `upgrade-k8s` command:

1. Updates API server, controller-manager, scheduler on CPs
2. Updates kube-proxy DaemonSet
3. Updates kubelet on all nodes (rolling, one at a time)
4. Waits for each component to be healthy before proceeding

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

Kubernetes follows semantic versioning. You can skip patch versions (1.32.0 → 1.32.2) but **not** minor versions (1.31 → 1.33). Always upgrade one minor version at a time: 1.31 → 1.32 → 1.33.

</div>
</div>

---

## Adding or Changing Extensions

Extensions (qemu-guest-agent, iscsi-tools, tailscale, etc.) are baked into the Talos image. Changing them requires a new schematic and an upgrade.

### Via Terraform (Preferred)

Update `terraform.tfvars`:

```hcl
talos_extensions = ["qemu-guest-agent", "iscsi-tools"]
```

```bash
terraform plan   # See the new schematic ID
terraform apply  # Downloads new image, but doesn't upgrade running nodes
```

Terraform updates the image and machine configs, but running nodes keep their current version. You still need to `talosctl upgrade` each node with the new schematic.

### Via Image Factory (Manual)

1. Go to [factory.talos.dev](https://factory.talos.dev/)
2. Select your Talos version
3. Pick the extensions you want
4. Copy the schematic ID
5. Upgrade each node with the new image URL

```bash
NEW_SCHEMATIC="<new-schematic-id>"
TALOS_VERSION="v1.9.4"

talosctl upgrade \
    --image factory.talos.dev/installer/${NEW_SCHEMATIC}:${TALOS_VERSION} \
    --preserve \
    --nodes 192.168.1.80
```

### Verify Extensions

```bash
talosctl get extensions --nodes <NODE_IP>
```

---

## Rollback

### Talos Rollback

Talos keeps the previous OS image on a secondary partition. If an upgrade breaks the node, rollback:

```bash
talosctl rollback --nodes <NODE_IP>
```

The node reboots into the previous Talos version. This works because Talos uses an A/B partition scheme — the upgrade writes to partition B while A stays intact. Rollback switches the boot target.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

Rollback only works for the **immediately previous** version. If you upgrade v1.9.2 → v1.9.4 → v1.9.5, you can only roll back to v1.9.4, not v1.9.2. To go further back, use `talosctl upgrade` to the older version explicitly.

</div>
</div>

### Kubernetes Rollback

There's no built-in `talosctl rollback-k8s`. To revert a Kubernetes upgrade:

```bash
talosctl upgrade-k8s --to 1.32.0 --nodes 192.168.1.70
```

This works because `upgrade-k8s` doesn't care about direction — it sets the target version and converges.

### Nuclear Option: Rebuild

If the cluster is unrecoverable after an upgrade, don't be precious about it. Rebuild.

1. `terraform destroy` — Remove all VMs
2. Revert `talos_version` in `terraform.tfvars` to the known-good version
3. `terraform apply` — Fresh cluster
4. [Restore from Velero](/2026-02-08-k8s-velero-backups/#disaster-recovery-new-cluster)

The whole cycle — destroy, rebuild, restore — takes about 30 minutes with Terraform and Velero. That's why you take backups before upgrading.

---

## Staying Current

### Version Monitoring

I check for updates monthly. Talos releases roughly every 2–3 weeks.

```bash
# Check current versions
talosctl version --nodes 192.168.1.70 --short
kubectl version --short

# Check latest Talos release
curl -s https://api.github.com/repos/siderolabs/talos/releases/latest | jq -r '.tag_name'
```

### Upgrade Cadence

| Release type | How often | Risk | Strategy |
|-------------|-----------|------|----------|
| Talos patch (v1.9.x) | Every 2–3 weeks | Low | Apply within a week |
| Talos minor (v1.x.0) | Every 3–4 months | Medium | Wait for .1 or .2 patch, then upgrade |
| Kubernetes patch (1.32.x) | Monthly | Low | Apply within 2 weeks |
| Kubernetes minor (1.x.0) | Every 4 months | Medium | Wait for .1 patch, read changelog |

Don't let versions rot. But don't rush a .0 release into your cluster the day it drops either. Let others find the edge cases.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Subscribe to Talos releases on GitHub (Watch → Custom → Releases). Reading the changelog takes 5 minutes and prevents surprises.

</div>
</div>

---

## Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `upgrade` hangs at "waiting for node" | Node didn't reboot cleanly | Check Proxmox console, force reboot VM |
| Node boots but doesn't rejoin | etcd member stale after upgrade | `talosctl etcd remove-member`, node re-adds on boot |
| `upgrade-k8s` fails with version error | Skipped a minor version | Upgrade one minor at a time (1.31→1.32→1.33) |
| Extensions missing after upgrade | Used wrong schematic ID | Check `talosctl get extensions`, re-upgrade with correct schematic |
| Pods stuck Terminating after reboot | Node drain timed out | `kubectl delete pod --grace-period=0 --force` |
| `--preserve` didn't preserve data | Disk path changed between versions | Rare. Check release notes for disk handling changes |
| API unreachable during CP upgrade | Single CP — expected | Wait 30–90 seconds. Consider HA (3 CPs) |

---

## What I Learned

### 1. Workers First Is Not Optional

I upgraded the control plane first once. The CP rebooted with a new kubelet version. The workers were running the old version. kubelet on the CP couldn't communicate with kubelets on the workers — API version mismatch. The API server was up but reported all workers as `NotReady`. Every pod went to `Pending`. Plex down. Family riot.

Workers first means the CP is always running a version equal to or older than the workers. Older CPs can talk to newer workers. Newer CPs with older workers — version skew bites.

### 2. --preserve Saves Hours

The first time I upgraded without `--preserve`, I lost all local PVCs. Talos reformatted the data partition as part of the "clean" upgrade. Sonarr config — gone. Prowlarr indexers — gone. Rebuilt from Velero, but Restic restores aren't fast. `--preserve` is almost always what you want. The only exception is if the release notes say the partition format changed.

### 3. The Dashboard Is Your Best Friend

`talosctl dashboard` during an upgrade gives you real-time visibility: CPU, memory, disk, network, service status, kernel logs — all in one TUI. Way better than running five commands in parallel.

```bash
talosctl dashboard --nodes 192.168.1.70,192.168.1.80,192.168.1.81
```

When a node reboots, you see it disappear and come back. When services start, you see them go green one by one. It's the difference between "I hope it's working" and "I can see it working."

### 4. Read the Release Notes

Skipped the release notes for Talos v1.9.0 because "it's just a patch." It wasn't — it was a minor release that changed how machine configs handle network bonds. The upgrade succeeded, but the next `talosctl apply-config` failed with a schema error I didn't understand for 30 minutes.

Five minutes of reading prevents thirty minutes of debugging. The Talos team writes excellent release notes. Respect the changelog.

---

## What's Next

- [Talos Day-2 Operations](/2026-02-11-k8s-talos-operations/) — etcd management, node replacement, config patches, troubleshooting
- [Deploy the cluster](/2026-02-08-k8s-talos-proxmox-deploy/) — If you haven't built the cluster yet
- [Velero Backups](/2026-02-08-k8s-velero-backups/) — Disaster recovery before you need it
- [Cluster Autoscaling](/2026-02-11-k8s-talos-autoscaling/) — HPA, VPA, and node autoscaling for Talos

---

## References

- [Talos Upgrade Guide](https://www.talos.dev/latest/talos-guides/upgrading-talos/)
- [Kubernetes Upgrade Guide (Talos)](https://www.talos.dev/latest/kubernetes-guides/upgrading-kubernetes/)
- [Talos Support Matrix](https://www.talos.dev/latest/introduction/support-matrix/)
- [Talos Image Factory](https://factory.talos.dev/)
- [Talos Releases & Changelogs](https://github.com/siderolabs/talos/releases)
- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
