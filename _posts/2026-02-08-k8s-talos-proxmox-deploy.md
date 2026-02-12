---
title: "Deploying Kubernetes on Proxmox with Terraform and Talos"
subtitle: "Talos Linux Kubernetes cluster on Proxmox VE via Terraform. Image factory, VM provisioning, bootstrap, credentials - one apply."
date: 2026-02-08
tags:
  - kubernetes
  - talos
  - proxmox
  - terraform
  - homelab
  - getting-started
categories:
  - Kubernetes Homelab
readtime: true
---
Talos Linux Kubernetes cluster on Proxmox VE via Terraform. Image factory, VM provisioning, bootstrap, credentials - one `terraform apply`.
<!--more-->

> *"Just SSH in and fix it manually"* - Things you'll never say again with Talos

---

## Problem

I've built Kubernetes clusters three different ways. The first time with kubeadm, I documented nothing and forgot half the steps. The second time with Ansible, the playbooks broke on upgrades. The third time I did it right.

You want a cluster you can destroy and rebuild in 15 minutes. One that doesn't depend on SSH-ing into nodes and running commands you barely remember.

## Solution

[Talos Linux](https://www.talos.dev/) + Terraform. Talos is immutable and API-only - no SSH, no shell, no "let me just fix this one thing." Either it's in your Terraform config or it doesn't exist. This sounds limiting until you need to rebuild and realize you can.

Terraform manages the VMs. Talos manages everything inside them. One `terraform apply` gives you a working cluster.

Full source: [k8s-deploy](https://github.com/YOUR-USERNAME/k8s-deploy)

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Homelab defaults</div>
<div class="admonition-content" markdown="1">

This uses a single control plane and flat networking. Fine for a homelab. For production, you need 3+ control planes, separate VLANs, and remote state with locking. I'll point out the differences.

</div>
</div>

---

## Repo Structure

```
├── versions.tf          # Terraform + provider version pins
├── providers.tf         # Proxmox and Talos provider config
├── variables.tf         # Inputs with defaults
├── locals.tf            # Computed IPs, node maps, cluster endpoint
├── image.tf             # Talos image factory + download
├── main.tf              # VM resources (control plane + workers)
├── talos.tf             # Machine config, bootstrap, health check, creds
├── outputs.tf           # kubeconfig, talosconfig, node IPs
├── terraform.tfvars.example
└── generated/           # (git-ignored) kubeconfig + talosconfig
```

---

## Providers

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = ">= 0.69.0"
    }
    talos = {
      source  = "siderolabs/talos"
      version = ">= 0.7.0"
    }
    local = {
      source  = "hashicorp/local"
      version = ">= 2.0.0"
    }
  }
}
```

[bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs) manages VMs.
[siderolabs/talos](https://registry.terraform.io/providers/siderolabs/talos/latest/docs) handles
image factory, machine configs, bootstrap, and health checks.

The Proxmox provider needs both API and SSH access. API for VM operations, SSH for uploading the Talos image to Proxmox storage. This confused me at first - why SSH? Because the image upload happens via SCP under the hood.

```hcl
provider "proxmox" {
  endpoint  = var.proxmox_endpoint
  insecure  = var.proxmox_insecure
  api_token = var.proxmox_api_token

  ssh {
    agent    = true
    username = var.proxmox_ssh_user
  }
}
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

`proxmox_insecure = true` works around self-signed certs. I use it because regenerating certs on my homelab Proxmox every year is annoying. Production needs proper certs.

</div>
</div>

---

## Variables

Only two are required. Everything else has defaults.

```hcl
# Required
proxmox_endpoint  = "https://<PROXMOX_IP>:8006"
proxmox_api_token = "<USER>@pam!<TOKEN_NAME>=<TOKEN_SECRET>"
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Create a dedicated API token: Datacenter → Permissions → API Tokens. `PVEVMAdmin` on `/`.
Don't use root.

</div>
</div>

Key defaults:

| Variable | Default | Notes |
|----------|---------|-------|
| `cluster_name` | `"talos"` | Prefixed to VM names |
| `talos_version` | `"v1.9.2"` | Pinned for reproducibility |
| `controlplane_count` | `1` | Set to 3 + `cluster_vip` for HA |
| `worker_count` | `2` | Scale by changing this |
| `network_cidr` | `"10.0.0.0/24"` | IPs calculated from offsets |
| `cp_ip_offset` | `70` | First CP gets .70 |
| `worker_ip_offset` | `80` | First worker gets .80 |

### How IPs Are Computed

The first time I did this, I hardcoded IPs in every resource. Scaling from 2 to 3 workers meant editing six places. Missed one, spent 20 minutes figuring out why the third worker wouldn't join.

Use maps. Compute everything once:

```hcl
locals {
  controlplanes = {
    for i in range(var.controlplane_count) :
    "${var.cluster_name}-cp-${i + 1}" => {
      vm_id = var.vm_id_base + i
      ip    = cidrhost(var.network_cidr, var.cp_ip_offset + i)
    }
  }

  workers = {
    for i in range(var.worker_count) :
    "${var.cluster_name}-w-${i + 1}" => {
      vm_id = var.vm_id_base + 10 + i
      ip    = cidrhost(var.network_cidr, var.worker_ip_offset + i)
    }
  }
}
```

Change `worker_count` from 2 to 4? Terraform adds two nodes with correct IPs, VM IDs, and hostnames. One variable change.

---

## Image Factory

Talos [image factory](https://factory.talos.dev/) builds custom OS images with the extensions you
specify. This downloads a `nocloud` image with `qemu-guest-agent` baked in:

```hcl
data "talos_image_factory_extensions_versions" "this" {
  talos_version = var.talos_version
  filters = { names = var.talos_extensions }
}

resource "talos_image_factory_schematic" "this" {
  schematic = yamlencode({
    customization = {
      systemExtensions = {
        officialExtensions = data.talos_image_factory_extensions_versions.this.extensions_info[*].name
      }
    }
  })
}

resource "proxmox_virtual_environment_download_file" "talos" {
  content_type            = "iso"
  datastore_id            = var.image_storage
  node_name               = var.proxmox_node
  file_name               = "talos-${var.talos_version}-${var.cluster_name}.img"
  url                     = "${var.talos_factory_url}/image/${talos_image_factory_schematic.this.id}/${var.talos_version}/nocloud-amd64.raw.gz"
  decompression_algorithm = "gz"
  overwrite               = false
  overwrite_unmanaged     = true
}
```

Terraform downloads this once. Subsequent applies skip the download if the file exists.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Start with just `qemu-guest-agent`. Add other extensions when you need them. I added `iscsi-tools` when I connected to a SAN, `tailscale` when I wanted remote access. Don't preemptively install things you might need someday.

</div>
</div>

---

## VM Provisioning

Both roles use the same module, differing only in sizing and tags:

```hcl
module "controlplane" {
  source   = "git::https://github.com/YOUR-USERNAME/terraform-proxmox.git"
  for_each = local.controlplanes

  vm_id     = each.value.vm_id
  name      = each.key
  node_name = var.proxmox_node
  tags      = [var.cluster_name, "controlplane", "talos"]
  on_boot   = true

  disk_image_id = proxmox_virtual_environment_download_file.talos.id
  disk_size_gb  = var.controlplane_disk_gb
  disk_storage  = var.disk_storage
  boot_order    = ["scsi0"]

  cpu_cores     = var.controlplane_cpu
  memory_mb     = var.controlplane_memory_mb
  agent_enabled = local.has_guest_agent

  network_bridge = var.network_bridge
  vlan_id        = var.vlan_id

  initialize                  = true
  initialization_datastore_id = var.disk_storage
  ip_address                  = "${each.value.ip}/${local.network_prefix}"
  gateway                     = var.gateway
  dns                         = var.nameservers
}
```

Workers: same module, `local.workers`, worker sizing.

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

VM module lives in a separate
[terraform-proxmox](https://github.com/YOUR-USERNAME/terraform-proxmox) repo for reuse across
projects.

</div>
</div>

Default sizing:

| Role | CPU | Memory | Disk | VM ID base |
|------|-----|--------|------|------------|
| Control Plane | 2 | 4 GB | 20 GB | 400 |
| Worker | 2 | 4 GB | 50 GB | 410 |

---

## Cluster Configuration and Bootstrap

After VMs boot, Terraform configures them via the Talos API (port 50000). No SSH.

### Secrets

```hcl
resource "talos_machine_secrets" "this" {
  talos_version = var.talos_version
}
```

Generates cluster PKI (CA, certs, tokens). Stored in Terraform state.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

State contains cluster secrets. Use remote state with encryption for anything beyond a lab.

</div>
</div>

### Machine Configs

```hcl
data "talos_machine_configuration" "controlplane" {
  cluster_name     = var.cluster_name
  cluster_endpoint = "https://${local.cluster_endpoint}:6443"
  machine_type     = "controlplane"
  machine_secrets  = talos_machine_secrets.this.machine_secrets
  talos_version    = var.talos_version
}

resource "talos_machine_configuration_apply" "controlplane" {
  depends_on = [module.controlplane]
  for_each   = local.controlplanes

  client_configuration        = talos_machine_secrets.this.client_configuration
  machine_configuration_input = data.talos_machine_configuration.controlplane.machine_configuration
  node                        = each.value.ip
  config_patches = [
    yamlencode({
      machine = {
        network = { hostname = each.key }
        install = { disk = var.install_disk }
      }
    })
  ]
}
```

Base config generated from cluster name + role + secrets. Per-node patches set hostname and
install disk in a single `yamlencode` block. Workers identical with `machine_type = "worker"`.

### Client Configuration

```hcl
data "talos_client_configuration" "this" {
  cluster_name         = var.cluster_name
  client_configuration = talos_machine_secrets.this.client_configuration
  endpoints            = [for _, cp in local.controlplanes : cp.ip]
}
```

Generates the `talosconfig` used by `talosctl` and referenced by the health check.

### Bootstrap

```hcl
resource "talos_machine_bootstrap" "this" {
  depends_on           = [talos_machine_configuration_apply.controlplane]
  node                 = local.first_cp_ip
  endpoint             = local.first_cp_ip
  client_configuration = talos_machine_secrets.this.client_configuration
}
```

> *"Is it done yet?"* - Just wait for the health check instead of refreshing frantically

Runs once on the first control plane node. Initializes etcd and the Kubernetes API.

### Health Check

```hcl
data "talos_cluster_health" "this" {
  depends_on = [
    talos_machine_bootstrap.this,
    talos_machine_configuration_apply.controlplane,
    talos_machine_configuration_apply.worker,
  ]

  client_configuration = data.talos_client_configuration.this.client_configuration
  control_plane_nodes  = [for _, cp in local.controlplanes : cp.ip]
  worker_nodes         = [for _, w in local.workers : w.ip]
  endpoints            = data.talos_client_configuration.this.endpoints

  timeouts = { read = "10m" }
}
```

Blocks until all nodes are healthy or 10 minutes elapse. If a node doesn't join, `apply` fails
explicitly instead of producing a half-working cluster.

---

## Credentials

> *"Treat your infrastructure like cattle, not pets. Except for the credentials - guard those like the Crown Jewels."* - Cloud Native Wisdom

```hcl
resource "local_sensitive_file" "kubeconfig" {
  depends_on = [data.talos_cluster_health.this]

  content         = talos_cluster_kubeconfig.this.kubeconfig_raw
  filename        = "${path.module}/generated/kubeconfig"
  file_permission = "0600"
}

resource "local_sensitive_file" "talosconfig" {
  depends_on = [data.talos_cluster_health.this]

  content         = data.talos_client_configuration.this.talos_config
  filename        = "${path.module}/generated/talosconfig"
  file_permission = "0600"
}
```

Written to `generated/` (0600, git-ignored). Or pull from outputs:

```bash
terraform output -raw kubeconfig > ~/.kube/talos.kubeconfig
terraform output -raw talosconfig > ~/.talos/config
```

---

## Usage

### Deploy

```bash
cp terraform.tfvars.example terraform.tfvars
# Set proxmox_endpoint and proxmox_api_token

terraform init
terraform plan
terraform apply
```

The apply sequence:

1. Build Talos schematic, download image to Proxmox
2. Create control plane + worker VMs
3. Generate secrets, apply machine configs via Talos API
4. Bootstrap etcd + Kubernetes API on first control plane
5. Wait for all nodes healthy (up to 10m)
6. Write kubeconfig + talosconfig to `generated/`

### Verify

```bash
export KUBECONFIG=generated/kubeconfig
kubectl get nodes

export TALOSCONFIG=generated/talosconfig
talosctl health
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

`talosctl dashboard` gives you a real-time TUI showing CPU, memory, network, and service
status across all nodes. It's the fastest way to confirm the cluster is healthy after a fresh
deploy or upgrade:
```bash
talosctl dashboard --nodes <CP_IP>,<W1_IP>,<W2_IP>
```
For a single node deep-dive, `talosctl dmesg -n <NODE_IP> --follow` streams kernel logs in
real time - useful for debugging boot issues or hardware problems.

</div>
</div>

### Scale Workers

```hcl
worker_count = 4
```

`terraform apply`. New VMs created, configured, joined. Existing nodes untouched.

### HA Control Plane

```hcl
controlplane_count = 3
cluster_vip        = "10.0.0.69"
```

The VIP floats between control plane nodes. API stays reachable if a node goes down.

### Upgrade Talos

```bash
terraform output talos_schematic_id

talosctl upgrade \
  --image factory.talos.dev/installer/<SCHEMATIC_ID>:<NEW_VERSION> \
  --preserve \
  --nodes <NODE_IP>
```

Schematic ID preserves your extensions. `--preserve` keeps the data partition. Upgrade workers
first, control plane last.

### Tear Down

```bash
terraform destroy
```

---

## Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `apply` hangs on machine config | VM didn't boot or wrong IP | Check Proxmox console, verify network/VLAN |
| Health check times out | Node can't reach API server | Check firewall rules between nodes, verify `cluster_endpoint` |
| `connection refused` on port 50000 | Talos not ready yet | Wait. First boot takes 1-2 minutes. Re-run `apply`. |
| Image download fails | Proxmox can't reach factory.talos.dev | Check DNS and outbound HTTPS on Proxmox node |

---

## What's Next

You have a working Kubernetes cluster. Next steps:

- [Set up the foundation layer](/2026-02-08-k8s-homelab-infra/) - MetalLB for LoadBalancer IPs,
  Traefik for ingress routing, NFS CSI for persistent storage
- [Deploy a media stack](/2026-02-08-k8s-media-stack/) - Plex, Sonarr, Radarr, and friends on top
  of that infrastructure using Helm
- [Migrate to Flux GitOps](/2026-02-08-k8s-flux-gitops/) - automated delivery from Git, no more
  manual `helm upgrade`

The cluster is API-managed end to end, so everything stays declarative.

---

## References

- [Source code](https://github.com/YOUR-USERNAME/k8s-deploy)
- [Talos Linux docs](https://www.talos.dev/latest/)
- [Talos Terraform provider](https://registry.terraform.io/providers/siderolabs/talos/latest/docs)
- [bpg/proxmox Terraform provider](https://registry.terraform.io/providers/bpg/proxmox/latest/docs)
- [Talos Image Factory](https://factory.talos.dev/)
- [Talos system extensions](https://github.com/siderolabs/extensions)
