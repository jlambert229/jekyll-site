---
title: "Terraform and vSphere - mk3"
subtitle: "Packaging the mk2 vSphere deployment into a reusable Terraform module."
date: 2022-01-26
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
Packaging the mk2 vSphere deployment into a reusable Terraform module.
<!--more-->

---

## Problem

The [mk2](/2022-01-19-terraform-vsphere-vm-mk2/) deployment uses variables, but the code lives in one
project. If another team wants to use it, or you want to deploy a VM alongside other resources
(DNS records, load balancers), you need to copy/paste. Modules solve this.

## What Modules Are

A Terraform module is a directory of `.tf` files. The directory you run `terraform` from is the
*root module*. Any directory you reference via `source` is a *child module*.

> *"Modules are reusable units of infrastructure. Write once, deploy everywhere. DRY applies to infra too."* - IaC principle

*"Shiny."* - Firefly. Modular Terraform is shiny. Write once, reuse everywhere. No more copy-paste archaeology. Source: [vsphere-vm-mk3](https://github.com/YOUR-USERNAME/vsphere-vm-mk3)

---

## File Structure

```
.
├── deployment.tf          # Calls the module, defines root outputs
├── root_variables.tf      # Variable declarations for tfvars
├── terraform.tfvars       # Input values
└── modules/
    └── vsphere-vm-mk3/
        ├── main.tf        # Resources (same as mk2)
        ├── variables.tf   # Module input variables
        ├── outputs.tf     # Module outputs
        ├── versions.tf    # Provider versions
        ├── README.md
        └── LICENSE
```

The mk2 files move into `modules/vsphere-vm-mk3/`. The root module calls into them.

---

## The Module Call

`deployment.tf` references the module, passes variables through, and exposes outputs:

```hcl
module "vsphere-vm-mk3" {
  source = "./modules/vsphere-vm-mk3"

  vsphere_user                 = var.vsphere_user
  vsphere_password             = var.vsphere_password
  vsphere_server               = var.vsphere_server
  vsphere_allow_unverified_ssl = var.vsphere_allow_unverified_ssl
  vsphere_datacenter           = var.vsphere_datacenter
  vsphere_datastore            = var.vsphere_datastore
  vsphere_compute_cluster      = var.vsphere_compute_cluster
  vsphere_network              = var.vsphere_network
  vsphere_template             = var.vsphere_template
  vm_name                      = var.vm_name
  vm_domain_name               = var.vm_domain_name
  vm_num_cpus                  = var.vm_num_cpus
  vm_memory                    = var.vm_memory
  vm_ipv4_address              = var.vm_ipv4_address
  vm_ipv4_netmask              = var.vm_ipv4_netmask
  vm_ipv4_gateway              = var.vm_ipv4_gateway
  vm_dns_server_list           = var.vm_dns_server_list
  vm_dns_suffix_list           = var.vm_dns_suffix_list
}

output "vm_ip_address" {
  value = module.vsphere-vm-mk3.vm_ip_address
}

output "vm_hostname" {
  value = module.vsphere-vm-mk3.vm_hostname
}
```

Module outputs are accessed via `module.<MODULE_NAME>.<OUTPUT_NAME>`.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Run `terraform init` after adding or changing a module source. Terraform needs to
initialize the module before it can plan or apply.

</div>
</div>

---

## Why Modules Matter

| Without Modules | With Modules |
|-----------------|--------------|
| Copy/paste code between projects | `source = "./modules/..."` or a git URL |
| Changes require editing every copy | Change the module once, all consumers get the update |
| Hard to compose resources | Call multiple modules from one root |

Modules also make it possible to publish to the [Terraform Registry](https://registry.terraform.io/)
for public or private sharing.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

When sourcing modules from git, use a `ref` tag for versioning:
`source = "git::https://github.com/org/repo.git//modules/vm?ref=v1.0.0"`. Without a ref,
Terraform pulls the latest commit on the default branch, which can break your deployment if the
module changes.

</div>
</div>

---

## Supplemental Files

### README

Document your module inputs, outputs, and usage. [terraform-docs](https://github.com/terraform-docs/terraform-docs)
can auto-generate this from your variable and output blocks.

### LICENSE

Required if sharing the module. Tells consumers how they can use your code.
See [opensource.guide/legal](https://opensource.guide/legal/) for help choosing a license.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Auto-generate module documentation from your code with
[terraform-docs](https://github.com/terraform-docs/terraform-docs):
```bash
terraform-docs markdown table ./modules/vsphere-vm-mk3/ > ./modules/vsphere-vm-mk3/README.md
```
It reads your `variables.tf` and `outputs.tf` and produces a clean table of inputs, outputs,
types, defaults, and descriptions. Run it in CI so the docs never drift from the code.

</div>
</div>

---

## What's Next

The VM module is reusable and composable. From here you can publish it to a private Terraform
registry, add CI with `terraform validate` and `tflint`, or layer it into larger deployments
alongside DNS, load balancers, and monitoring resources.

---

## References

- [Terraform modules tutorial](https://developer.hashicorp.com/terraform/tutorials/modules)
- [Module development docs](https://developer.hashicorp.com/terraform/language/modules/develop)
- [Module composition best practices](https://developer.hashicorp.com/terraform/language/modules/develop/composition)
