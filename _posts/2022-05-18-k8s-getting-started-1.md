---
title: "Deploying Kubernetes with Kubeadm and Calico - Part 1"
subtitle: "System prerequisites for a kubeadm Kubernetes cluster: swap, kernel modules, sysctl, containerd, and repository setup."
date: 2022-05-18
tags:
  - kubernetes
  - containers
  - getting-started
categories:
  - Kubernetes with Kubeadm
readtime: true
---
System prerequisites for a kubeadm Kubernetes cluster: swap, kernel modules, sysctl, containerd, and repository setup.
<!--more-->

> *"How hard can it be to set up a Kubernetes cluster?"* - Famous last words before discovering kubeadm

![This is fine - Cluster setup edition](/assets/img/memes/this-is-fine.jpeg)

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

Originally written for Ubuntu 20.04 and Kubernetes 1.24. Updated for the `pkgs.k8s.io`
repository and signed keyring method. Always check the
[official kubeadm docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
for the latest instructions.

</div>
</div>

---

## Overview

This part is boring but critical. You're preparing the OS before installing Kubernetes. Skip a step and you'll debug cryptic errors for an hour. Ask me how I know.

> *"The boring parts are the foundations. Skip the boring parts and the interesting parts become impossible."* - SRE wisdom

Part 1: prep all nodes (control plane and workers). [Part 2](/post/k8s-getting-started-2/): actually build the cluster. *"Great shot, kid. That was one in a million."* - Star Wars. Skipping prep steps is the opposite: failure is almost guaranteed.

Lab setup: 3 Ubuntu VMs (2 vCPU, 4 GB RAM, 50 GB disk, static IPs). Adjust for your environment.

![k8s-lab-image](/assets/img/k8s-lab.png)

---

## Disable Swap

> *"Kubernetes is opinionated about swap for the same reason surgeons are opinionated about sterile instruments."* - Resource Management 101

Kubernetes requires swap disabled. It needs precise resource accounting to schedule pods. Swap breaks that because the kernel can page out pod memory without Kubernetes knowing. Scheduler thinks a pod is using 1GB, it's actually using 2GB on disk. Bad things happen.

Disable swap immediately:

```bash
sudo swapoff -a
```

Prevent swap from re-enabling on reboot by commenting it out of `/etc/fstab`:

```bash
sudo sed -e '/swap/ s/^#*/#/' -i /etc/fstab
```

Verify:

```bash
cat /etc/fstab
# /swap.img none swap sw 0 0
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

After disabling swap, confirm it's actually off with `free -h`. The `Swap` row should show
all zeros. If swap is still active after `swapoff -a`, check for zram devices:
`cat /proc/swaps`.

</div>
</div>

---

## Kernel Modules

Pod networking requires `overlay` and `br_netfilter` modules.

Persist across reboots:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Load them now:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Verify:

```bash
lsmod | grep overlay && lsmod | grep br_netfilter
```

---

## Sysctl Settings

Enable IP forwarding and bridge iptables processing:

```bash
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Without these, pod-to-pod traffic across nodes gets dropped by the kernel.

> *"Networking is the substrate everything else depends on. Get it wrong once, debug it forever."* - Cluster operators' lament

---

## Install Prerequisites

```bash
sudo apt-get update && sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

---

## Install Containerd

Kubernetes needs a container runtime. This guide uses
[containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md).

Add Docker's GPG key and repository (containerd is distributed from Docker's repo):

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install:

```bash
sudo apt-get update && sudo apt-get install -y containerd.io
```

### Configure Containerd

Generate the default config and enable `SystemdCgroup`. Kubernetes requires the systemd cgroup
driver for proper resource management.

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Verify:

```bash
grep SystemdCgroup /etc/containerd/config.toml
# SystemdCgroup = true
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

If `SystemdCgroup` is still `false` after the `sed` command, edit the file manually. The
kubelet will fail to start if the cgroup driver doesn't match.

</div>
</div>

Restart containerd:

```bash
sudo systemctl restart containerd
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Verify containerd is working before moving on. `crictl` talks directly to the container
runtime:
```bash
sudo crictl info | head -20
sudo crictl images
```
If `crictl` can't connect, check `sudo systemctl status containerd` for errors. Catching a
broken runtime now saves time versus debugging a failed `kubeadm init` later.

</div>
</div>

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

If using a different container runtime (CRI-O, Docker Engine), configure the appropriate
cgroup driver. See
[Kubernetes container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).

</div>
</div>

---

## Add the Kubernetes Repository

Download the signing key:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

Replace `v1.32` with the minor version you want. The `pkgs.k8s.io` repository is versioned
per minor release.

</div>
</div>

Add the repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" | \
    sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Verify:

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```

---

## Run All Steps on All Nodes

Every command in this article needs to run on **every** node in the cluster - control plane
and workers alike. *"All of this has happened before, and all of this will happen again."* - Battlestar Galactica. Run the same steps on every node. Consistency is survival. [Part 2](/post/k8s-getting-started-2/) covers the node-specific steps.

> *"Symmetry kills surprises. If all nodes start from the same baseline, failures are predictable."* - Kubernetes deployment principle

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

If you're setting up multiple nodes, consider scripting these steps or using a configuration
management tool like Ansible. Running the same 15 commands manually on 3+ machines is where
mistakes happen.

</div>
</div>

---

## Related

- [Part 2: Cluster init with kubeadm](/post/k8s-getting-started-2/) - Initialize controller, add workers, install Calico
- [K8s Homelab Infrastructure](/post/k8s-homelab-infra/) - MetalLB, Traefik, NFS CSI after your cluster is up
- [Talos + Terraform cluster](/post/k8s-talos-proxmox-deploy/) - Alternative: immutable nodes, no manual prep

## References

- [kubeadm installation guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Containerd getting started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [pkgs.k8s.io announcement](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
