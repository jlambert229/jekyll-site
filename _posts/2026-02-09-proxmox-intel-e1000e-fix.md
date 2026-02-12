---
title: "Fixing Intel e1000e NIC Failures on Proxmox"
subtitle: "Network drops, backup freezes, and hardware unit hangs - the Intel I217/I219 on recent Proxmox kernels has a regression. Here's the ethtool workaround that actually works."
date: 2026-02-09
tags:
  - proxmox
  - networking
  - homelab
  - intel
  - e1000e
  - troubleshooting
categories:
  - Homelab Infrastructure
readtime: true
---
> *"When the network dies at 2am, you learn to love ethtool. And kernel pinning. And backups."* - Homelab incident retrospective

*"Game over, man! Game over!"* - Aliens. When the NIC goes dark mid-backup, it feels that way. But there's a fix. Proxmox hosts with Intel I217/I219 NICs (e1000e driver) can lose network connectivity under load. Backups freeze. VMs go dark. Unplugging and replugging the cable sometimes brings it back. This is a kernel regression - not your hardware.
<!--more-->

> *"The network just... stopped. Ping works from the switch but not from my laptop."* - Every homelab operator at 2am

---

## Symptoms

| What you see | When it happens |
|--------------|-----------------|
| Network interface shows `UP` but no traffic | During or after backups |
| SSH hangs, web UI unreachable | Under sustained load (sync jobs, VM migrations) |
| `dmesg` shows `e1000e: Detected Hardware Unit Hang` | Sporadically |
| Unplug/replug cable restores connectivity | Until next heavy traffic |

Affected NICs: **Intel I217-LM**, **I219-V**, **I219-LM**, and similar onboard gigabit controllers. The `e1000e` kernel driver has regressions in Proxmox kernels **6.8.12-9-pve** and newer (including 6.17.x). Your PCI I350 or other Intel NICs using the `igb` driver are fine.

---

## Root Cause

A regression in the e1000e driver's hardware offload path. Under load, the TSO/GSO/GRO path can hang the NIC. The driver thinks the interface is up; the hardware has effectively frozen.

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

This has been reported across Proxmox 6.8.x, 8.x, and kernel 6.17.x. The [Proxmox forum thread](https://forum.proxmox.com/threads/proxmox-6-8-12-9-pve-kernel-has-introduced-a-problem-with-e1000e-driver-and-network-connection-lost-after-some-hours.164439/) has dozens of confirmations. Intel I210 (igb driver) does **not** have this issue.

</div>
</div>

---

## Two Fix Options

| Option | Approach | Pros | Cons |
|--------|----------|------|------|
| **A. ethtool workaround** | Disable offload features on the affected interface | Stays on latest kernel; widely confirmed | Slight CPU overhead |
| **B. Kernel pinning** | Boot from an older kernel (e.g. 6.8.12-8-pve) | No config change to NIC | Stuck on old kernel until upstream fix |

**Recommendation:** Try **Option A** first. It works for most people and lets you keep current security patches.

---

## Option A: ethtool Workaround (Recommended)

### Step 1: Identify the interface

SSH to your Proxmox host:

```bash
# Find the I217/I219 (e1000e) interface
lspci -k | grep -A3 -i "e1000e\|219\|217"

# Interface name (often eno1 or enp0s31f6)
ip link show
```

The interface in use will typically be `eno1` or `enp0s31f6`. In my case it was **eno1** on a Dell OptiPlex 7080 with I219-LM.

### Step 2: Apply immediately (no reboot)

```bash
sudo ethtool -K eno1 gso off gro off tso off tx off rx off rxvlan off txvlan off sg off
```

Replace `eno1` with your interface name. Connectivity stays up; the fix takes effect instantly.

### Step 3: Make it persistent

Edit `/etc/network/interfaces` and add a `post-up` hook under the interface block:

```bash
# /etc/network/interfaces
iface eno1 inet manual
    post-up ethtool -K eno1 gso off gro off tso off tx off rx off rxvlan off txvlan off sg off

auto vmbr0
iface vmbr0 inet static
    address 192.168.2.131/24
    gateway 192.168.2.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

If `eno1` is already a bridge port (as above), the `post-up` runs when the interface is brought up. Reboot to verify it survives a restart.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

On Proxmox, `eno1` is often the bridge slave for `vmbr0`. The `post-up` runs during `ifup vmbr0` before the bridge attaches. No need to modify the bridge block - just add the `post-up` to the physical interface.

</div>
</div>

### Step 4: Verify

```bash
sudo ethtool -k eno1 | grep -E 'generic-segmentation-offload|generic-receive-offload|tcp-segmentation-offload'
```

Expected output:

```
tcp-segmentation-offload: off
generic-segmentation-offload: off
generic-receive-offload: off
```

---

## Option B: Kernel Pinning

If the ethtool workaround doesn't help (or you prefer not to change NIC settings):

```bash
# List available kernels
proxmox-boot-tool kernel list

# Pin to previous working kernel (e.g. 6.8.12-8-pve)
proxmox-boot-tool kernel pin 6.8.12-8-pve

# One-time test (next boot only)
proxmox-boot-tool kernel pin 6.8.12-8-pve --next-boot

# Unpin when fixed upstream
proxmox-boot-tool kernel unpin
```

Reboot after pinning. You'll stay on that kernel until you unpin.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

`--next-boot` is temporary. Use `proxmox-boot-tool kernel pin <name>` without flags to make it permanent. Otherwise you'll boot back into the broken kernel after the next reboot.

</div>
</div>

---

## Why This Works

> *"Sometimes the fix is 'turn off the optimization.' The kernel has bugs. Workarounds are valid engineering."* - Kernel regression survivor

**The ethtool fix:** Hardware offloads (TSO, GSO, GRO) move packet segmentation and reassembly into the NIC. The e1000e driver's offload path has a bug - under load it can hang the hardware. Disabling those offloads forces the kernel to do the work in software. Slightly more CPU, but the NIC stays stable.

**Kernel pinning:** The regression was introduced in a specific kernel release. Older kernels don't have the bad code path. Pinning avoids the problem entirely at the cost of not getting newer kernel fixes.

---

## Real-World Result

On a Dell OptiPlex 7080 (I219-LM) running Proxmox 8.x with kernel 6.17.9-1-pve:

- **Before:** Network died during backups; required cable unplug/replug.
- **After:** ethtool workaround applied + persistent config. Backups run without interruption.

No kernel downgrade. No hardware replacement. One `post-up` line.

---

## Related

- [Automated Proxmox Backups with Proxmox Backup Server](/2026-02-08-proxmox-backup-server/) - When your backups *do* work, here's how to automate them
- [Proxmox Automated Install](/2026-02-07-proxmox-automated-install/) - Reproducible installs for new nodes

---

## References

- [Proxmox forum: e1000e network connection lost (6.8.12-9-pve)](https://forum.proxmox.com/threads/proxmox-6-8-12-9-pve-kernel-has-introduced-a-problem-with-e1000e-driver-and-network-connection-lost-after-some-hours.164439/)
- [ethtool(8) - Linux man page](https://man7.org/linux/man-pages/man8/ethtool.8.html)
- [Proxmox boot tool: kernel pinning](https://forum.proxmox.com/threads/how-to-pin-unpin-a-specific-kernel.111732/)
