---
title: "Terraform and vSphere - mk2"
subtitle: "Improving the mk1 vSphere deployment with variables, file organization, and tfvars."
date: 2022-01-19
tags:
  - getting-started
  - hashicorp
  - terraform
  - vsphere
  - vmware
categories:
  - Terraform and vSphere
readtime: true
---
Improving the mk1 vSphere deployment with variables, file organization, and tfvars.
<!--more-->

---

## Problem

The [mk1](/2022-01-14-terraform-vsphere-vm-mk1/) deployment works but everything is hardcoded.
Changing an IP, VM name, or CPU count means editing `main.tf` directly. Not reusable.

## Solution

> *"Hardcoded values are tomorrow's technical debt with interest."* - Infrastructure as Code Best Practice

*"I don't like it. I don't agree with it. But I accept it."* - Bruce, Jaws. Hardcoding works until it doesn't. Extract variables. Future-you will thank present-you.

Source: [vsphere-vm-mk2](https://github.com/YOUR-USERNAME/vsphere-vm-mk2)

---

## File Organization

```
.
├── main.tf            # Resources and data sources
├── variables.tf       # Variable declarations
├── outputs.tf         # Output definitions
├── versions.tf        # Terraform and provider versions
└── terraform.tfvars   # Variable values (git-ignored)
```

Single-file Terraform is fine for learning. For anything you plan to maintain or share,
split it.

> *"File organization is documentation. Future-you will thank present-you for variables.tf."* - Terraform maintainability

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Files named `*.auto.tfvars` are loaded automatically without being specified on the command
line. Useful for layering environment-specific values (e.g., `dev.auto.tfvars`,
`prod.auto.tfvars`) when combined with workspaces.

</div>
</div>

---

## Variables in the Resource Block

The mk2 resource block replaces hardcoded values with `var.*` references:

```hcl
resource "vsphere_virtual_machine" "vm" {
  name             = var.vm_name
  resource_pool_id = data.vsphere_compute_cluster.cluster.resource_pool_id
  datastore_id     = data.vsphere_datastore.datastore.id

  num_cpus = var.vm_num_cpus
  memory   = var.vm_memory
  guest_id = data.vsphere_virtual_machine.template.guest_id

  scsi_type = data.vsphere_virtual_machine.template.scsi_type

  network_interface {
    network_id   = data.vsphere_network.network.id
    adapter_type = data.vsphere_virtual_machine.template.network_interface_types[0]
  }

  disk {
    label            = "disk0"
    size             = data.vsphere_virtual_machine.template.disks.0.size
    eagerly_scrub    = data.vsphere_virtual_machine.template.disks.0.eagerly_scrub
    thin_provisioned = data.vsphere_virtual_machine.template.disks.0.thin_provisioned
  }

  clone {
    template_uuid = data.vsphere_virtual_machine.template.id

    customize {
      network_interface {
        ipv4_address = var.vm_ipv4_address
        ipv4_netmask = var.vm_ipv4_netmask
      }

      ipv4_gateway    = var.vm_ipv4_gateway
      dns_server_list = var.vm_dns_server_list
      dns_suffix_list = var.vm_dns_suffix_list

      linux_options {
        host_name = var.vm_name
        domain    = var.vm_domain
      }
    }
  }
  lifecycle {
    ignore_changes = [
      annotation,
      clone[0].template_uuid,
      clone[0].customize[0].dns_server_list,
      clone[0].customize[0].network_interface[0]
    ]
  }
}
```

---

## Variable Declarations

Each variable gets a type, description, and optional default in `variables.tf`:

```hcl
variable "vsphere_user" {
  type        = string
  description = "Username for vSphere API operations."
  default     = "admin"
}

variable "vsphere_password" {
  type        = string
  description = "Password for vSphere API operations."
  sensitive   = true
  default     = "password"
}

variable "vsphere_server" {
  type        = string
  description = "vCenter server for vSphere API operations."
  default     = "vc-01"
}

variable "vsphere_allow_unverified_ssl" {
  type        = bool
  description = "Disable SSL certificate verification."
  default     = true
}
```

### Type Keywords

| Type | Use |
|------|-----|
| `string` | Text values (names, IPs, hostnames) |
| `number` | Numeric values (CPU count, memory) |
| `bool` | Flags (`true`/`false`) |
| `list(string)` | Collections (DNS servers, suffixes) |

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Add `validation` blocks to variables to catch bad input early. For example, validate that
`vm_num_cpus` is at least 1 or that `vm_ipv4_address` matches an IP regex. Terraform will
reject the plan before anything is created.

</div>
</div>

---

## Outputs

Same as mk1, just in their own file:

```hcl
output "vm_ip_address" {
  value = vsphere_virtual_machine.vm.default_ip_address
}

output "vm_hostname" {
  value = vsphere_virtual_machine.vm.name
}
```

---

## tfvars

The `terraform.tfvars` file provides values for declared variables:

```hcl
vsphere_datacenter      = "Datacenter"
vsphere_datastore       = "tpa-lab-01-ds-01"
vsphere_compute_cluster = "lab"
vsphere_network         = "pg-v110"
vsphere_template        = "templates/packer/ubuntu/linux-ubuntu-20"
vm_name                 = "test-vm-01"
vm_domain_name          = "foggyclouds.net"
vm_num_cpus             = 2
vm_memory               = 1024
```

Full example: [terraform.tfvars](https://github.com/YOUR-USERNAME/vsphere-vm-mk2/blob/main/terraform.tfvars)

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

Never commit a populated `terraform.tfvars` to a public repository. Add it to `.gitignore`
and provide a `.tfvars.example` with placeholder values instead.

</div>
</div>

---

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

`terraform console` is a REPL for testing expressions before putting them in code. Use it to
verify variable interpolation, function behavior, or data source lookups:
```bash
terraform console
> var.vm_name
"test-vm-01"
> cidrhost("10.1.110.0/24", 185)
"10.1.110.185"
> length(var.vm_dns_server_list)
1
```
Saves time versus running `plan` repeatedly to check if an expression evaluates correctly.

</div>
</div>

---

## What's Next

Variables make the deployment reusable, but the code is still in one project.
[mk3](/2022-01-26-terraform-vsphere-vm-mk3/) packages it into a Terraform module for sharing and
composition.

---

## References

- [Terraform variable documentation](https://developer.hashicorp.com/terraform/language/values/variables)
- [vSphere provider docs](https://registry.terraform.io/providers/hashicorp/vsphere/latest/docs)
- [Terraform recommended practices](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices)
