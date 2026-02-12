---
title: "Automating a Proxmox VE Install"
subtitle: "A single USB stick that produces a fully configured Proxmox VE host: answer files, first-boot scripts, and a USB prep tool."
date: 2026-02-07
tags:
  - proxmox
  - homelab
  - automation
  - getting-started
readtime: true
---
A single USB stick that produces a fully configured Proxmox VE host with zero manual intervention.
<!--more-->

> *"Automate yourself out of a job."* - Every engineer, probably

---

## Why Automate?

> *"Manual work is technical debt. Automation is paying it down."* - Operations reducing toil

*"Life finds a way."* - Jurassic Park. So do typos, forgotten steps, and "what did I do last time?" Manual installs find *a* way. Not necessarily the right one. I installed Proxmox manually three times. Each time took 45 minutes and I forgot something different. First time: forgot to add my SSH key. Second time: configured the wrong network interface. Third time: I automated it.

Manual installs work. Once. Then your drive dies or you buy new hardware and you're reading install notes from six months ago trying to remember what "configure firewall - see wiki" actually meant.

Proxmox supports automated installs via answer files. Add a first-boot script for post-install config. Burn a USB. Boot. Wait 10 minutes. Done. Same result every time.

Repo: [proxmox-automated-install](https://github.com/YOUR-USERNAME/proxmox-automated-install)

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Homelab shortcuts</div>
<div class="admonition-content" markdown="1">

This uses passwordless sudo and patches the subscription nag. Fine for a homelab. Don't copy this to production. The automation pattern works anywhere - just change what it automates.

</div>
</div>

Three files handle everything from building the installer to configuring the host:

| File | Purpose |
|------|---------|
| `answer.toml` | Drives the unattended Proxmox installer |
| `first-boot.sh` | Runs once after install to configure the host |
| `prepare-usb.sh` | Bakes everything into a bootable USB |

---

## The Answer File

The answer file is TOML that the Proxmox installer reads directly from the ISO. No human input
required. The repo ships an `answer.toml.example`. Copy it and fill in your values.

### Global Settings

```toml
[global]
keyboard = "en-us"
country = "us"
fqdn = "your-host.local"
mailto = "root@your-host.local"
timezone = "America/New_York"

# Generate with: mkpasswd --method=sha-512 'your-password-here'
root-password-hashed = "<YOUR_SHA512_HASH>"

root-ssh-keys = [
    "ssh-ed25519 <YOUR_PUBLIC_KEY>",
]

reboot-on-error = false
reboot-mode = "reboot"
```

`root-password-hashed` takes a SHA-512 hash, so no plaintext passwords live in the file. The SSH key
grants immediate key-based root access after install.

Setting `reboot-on-error = false` keeps the installer visible on failure so you can inspect it.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

`mkpasswd` lives in the `whois` package on Debian/Ubuntu. Install it with `apt install whois` if
the command isn't found.

</div>
</div>

### Network

```toml
[network]
source = "from-answer"
cidr = "<MGMT_IP>/24"
dns = "1.1.1.1"
gateway = "<GATEWAY_IP>"

filter.ID_NET_NAME = "eno*"
```

`source = "from-answer"` bypasses DHCP. The `ID_NET_NAME` filter matches the NIC by its
predictable interface name: `eno*` for onboard, `enp*` for PCIe, `eth*` for legacy. This
prevents the installer from binding to the wrong interface on multi-NIC machines.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Not sure what your NIC is named? Boot any Linux live USB on the target machine and run `ip link`
to see the interface names.

</div>
</div>

### Disk Setup

```toml
[disk-setup]
filesystem = "ext4"
disk-list = ["<TARGET_DISK>"]

lvm.maxroot = 100
lvm.swapsize = 8
lvm.minfree = 0
```

`disk-list` names the target device explicitly. This is critical on multi-drive hosts where you
don't want the installer guessing. If device names shift between boots, use a model filter instead:
`filter.ID_MODEL = "Your Drive Model*"`.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Run `lsblk -d -o NAME,SIZE,MODEL` from a live USB to identify your target disk and its model
string.

</div>
</div>

The LVM layout: 100 GB root, 8 GB swap, everything else becomes a thin pool for VM disks.
`minfree = 0` allocates all remaining space rather than leaving it unallocated in the VG.

### First-Boot Hook

```toml
[first-boot]
source = "from-iso"
ordering = "fully-up"
```

`from-iso` means the script is embedded in the prepared ISO. `fully-up` means it runs after
Proxmox services are available. This is required because the script calls `pveum` and `pvesm`,
which need the API stack running.

---

## The First-Boot Script

Everything the answer file can't do lives here: repository switching, user creation, security
hardening, storage, and NTP. The script uses `set -euo pipefail` and logs every step to
`/var/log/first-boot-config.log`.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

If something goes wrong after install, check this log first:
`cat /var/log/first-boot-config.log`. It shows exactly which step failed and why.

</div>
</div>

### Switch to the No-Subscription Repository

```bash
# Disable enterprise repos - handles both .list (PVE 8) and .sources (PVE 9+)
for f in /etc/apt/sources.list.d/pve-enterprise.list /etc/apt/sources.list.d/ceph.list; do
    [ -f "$f" ] && sed -i 's/^deb/#deb/' "$f"
done
for f in /etc/apt/sources.list.d/pve-enterprise.sources /etc/apt/sources.list.d/ceph.sources; do
    if [ -f "$f" ]; then
        sed -i 's/^Enabled: yes/Enabled: no/' "$f"
        grep -q "^Enabled:" "$f" || sed -i '/^Types:/i Enabled: no' "$f"
    fi
done

# Add no-subscription repo
CODENAME=$(grep VERSION_CODENAME /etc/os-release | cut -d= -f2)
echo "deb http://download.proxmox.com/debian/pve ${CODENAME} pve-no-subscription" \
    > /etc/apt/sources.list.d/pve-no-subscription.list
```

Proxmox ships with enterprise repos enabled. Without a subscription, `apt update` fails. This
disables them and adds the community repo. It handles both `.list` (PVE 8) and `.sources`
(PVE 9+) formats, so the script works across versions.

<div class="admonition admonition-production" markdown="1">
<div class="admonition-title">🏭 In production</div>
<div class="admonition-content" markdown="1">

Use the enterprise repository with a valid subscription. It provides tested, stable updates and
access to Proxmox support.

</div>
</div>

### Install Baseline Packages

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y -qq \
    curl wget vim htop iotop net-tools dnsutils \
    gnupg lsb-release ca-certificates sudo rsync \
    tmux unzip tree jq nfs-common ufw fail2ban chrony
```

The tools needed for sysadmin work plus the services configured later in the script.

### Create an Admin User

```bash
useradd -m -s /bin/bash -G sudo "<ADMIN_USER>"

# SSH key
mkdir -p "/home/<ADMIN_USER>/.ssh"
echo "<SSH_PUBLIC_KEY>" > "/home/<ADMIN_USER>/.ssh/authorized_keys"
chmod 700 "/home/<ADMIN_USER>/.ssh"
chmod 600 "/home/<ADMIN_USER>/.ssh/authorized_keys"

# Passwordless sudo
echo "<ADMIN_USER> ALL=(ALL) NOPASSWD:ALL" > "/etc/sudoers.d/<ADMIN_USER>"
chmod 440 "/etc/sudoers.d/<ADMIN_USER>"

# Proxmox access
pveum user add "<ADMIN_USER>@pam"
pveum aclmod / -user "<ADMIN_USER>@pam" -role PVEAdmin
```

A dedicated admin user with SSH key access, passwordless sudo, and PVEAdmin in the Web UI.
Day-to-day work uses this named account (audit trail); root is reachable via `sudo` when needed.

<div class="admonition admonition-production" markdown="1">
<div class="admonition-title">🏭 In production</div>
<div class="admonition-content" markdown="1">

Avoid `NOPASSWD` sudo. Require password confirmation for privilege escalation and scope the PVE
role more tightly than PVEAdmin based on actual needs.

</div>
</div>

### Harden SSH

```bash
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^#\?X11Forwarding.*/X11Forwarding no/' /etc/ssh/sshd_config

systemctl restart sshd
```

Keys only, no passwords. Root can still authenticate with a key (`prohibit-password`).
X11 forwarding disabled because a headless hypervisor has no use for it.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Before closing your current session, open a second SSH connection to verify key auth works.
Locking yourself out of a headless machine means pulling out a monitor and keyboard.

</div>
</div>

### Configure NTP

> *"Time synchronization isn't sexy, but neither is debugging why your distributed system randomly fails."* - Infrastructure Engineering

```bash
cat > /etc/chrony/chrony.conf <<'CHRONY'
pool 0.pool.ntp.org iburst
pool 1.pool.ntp.org iburst
pool 2.pool.ntp.org iburst
pool 3.pool.ntp.org iburst

driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
CHRONY

systemctl enable chrony
systemctl restart chrony
```

Accurate time matters more on a hypervisor than almost anywhere else. VM clocks derive from the
host, certificate validation depends on it, and cluster operations break without it. `iburst`
speeds up the initial sync after a fresh boot.

### Configure the Firewall

```bash
ufw --force reset
ufw default deny incoming
ufw default allow outgoing

ufw allow 22/tcp comment 'SSH'
ufw allow 8006/tcp comment 'Proxmox Web UI'
ufw allow 3128/tcp comment 'SPICE Proxy'
ufw allow 5900:5999/tcp comment 'VNC for VMs'
ufw allow 111/udp comment 'NFS rpcbind'

ufw --force enable
```

Default deny with an explicit allowlist. Only management (SSH, Web UI), VM console (SPICE, VNC),
and NFS traffic gets through.

<div class="admonition admonition-production" markdown="1">
<div class="admonition-title">🏭 In production</div>
<div class="admonition-content" markdown="1">

Consider using Proxmox's built-in firewall (pve-firewall) instead of UFW. It integrates with the
cluster config, supports per-VM rules, and is manageable through the Web UI and API. Also restrict
source IPs for SSH and the Web UI to management VLANs.

</div>
</div>

### Configure fail2ban

```bash
cat > /etc/fail2ban/jail.local <<'F2B'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log

[proxmox]
enabled = true
port = https,8006
filter = proxmox
backend = systemd
F2B

cat > /etc/fail2ban/filter.d/proxmox.conf <<'F2BF'
[Definition]
failregex = pvedaemon\[.*authentication (verification )?failure; rhost=<HOST> user=\S+ msg=.*
ignoreregex =
journalmatch = _SYSTEMD_UNIT=pvedaemon.service
F2BF
```

Two jails: SSH and the Proxmox Web UI. Five failures in 10 minutes triggers a one-hour ban. The
Proxmox filter watches `pvedaemon.service` via the systemd journal.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Locked yourself out? From the console, run `fail2ban-client set sshd unbanip <YOUR_IP>` to unban
immediately. Run `fail2ban-client status sshd` to see who's currently banned.

</div>
</div>

### Add NFS Storage

```bash
pvesm add nfs "<NFS_STORAGE_NAME>" \
    --server "<NFS_SERVER_IP>" \
    --export "<NFS_EXPORT_PATH>" \
    --path "/mnt/pve/<NFS_STORAGE_NAME>" \
    --content "images,iso,backup,snippets,vztmpl" \
    --options "vers=3,soft,intr"
```

Adds an NFS share as a Proxmox storage backend. The `content` flag controls what types of data
can live there: disk images, ISOs, backups, snippets, and container templates. The `soft,intr`
mount options mean NFS operations time out and return errors rather than hanging indefinitely if
the server goes offline.

<div class="admonition admonition-production" markdown="1">
<div class="admonition-title">🏭 In production</div>
<div class="admonition-content" markdown="1">

Use NFS v4 with Kerberos authentication, or dedicated storage networks (iSCSI, Ceph) with
redundancy. A single NAS is a single point of failure for all VM storage and backups.

</div>
</div>

### Remove the Subscription Nag

> *"I'll just click OK every time"* - You won't, but the script will handle it anyway

```bash
NAG_FILE="/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js"
if [ -f "$NAG_FILE" ] && grep -q "Ext.Msg.show" "$NAG_FILE"; then
    cp "$NAG_FILE" "${NAG_FILE}.bak"
    sed -Ei "s/^\s*(Ext\.Msg\.show\(\{)$/void({ \/\/ \1/" "$NAG_FILE"
    systemctl restart pveproxy
fi
```

Patches out the subscription reminder dialog. A backup of the original file is created first.
This is purely a lab convenience.

<div class="admonition admonition-production" markdown="1">
<div class="admonition-title">🏭 In production</div>
<div class="admonition-content" markdown="1">

Purchase a Proxmox subscription. You get stable enterprise repos, direct support, and you fund the
project that makes all of this possible. Don't patch this out in production environments.

</div>
</div>

---

## The USB Prep Script

The glue. Takes a stock Proxmox ISO, embeds the answer file and first-boot script, and writes the
result to a USB drive.

```bash
sudo ./prepare-usb.sh <proxmox-iso> <usb-device>
# Example: sudo ./prepare-usb.sh ~/Downloads/proxmox-ve_8.3-1.iso /dev/sda
```

The script:

1. **Validates dependencies:** `xorriso`, `dd`, `proxmox-auto-install-assistant`
2. **Validates inputs:** ISO exists, USB is a block device, password isn't the placeholder
3. **Validates the answer file:** runs `proxmox-auto-install-assistant validate-answer`
4. **Prepares the ISO:** embeds answer file + first-boot script via `prepare-iso`
5. **Writes to USB:** `dd` with 4M blocks, confirmation prompt before destroying data
6. **Cleans up:** removes the temp ISO from `/tmp`

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

`proxmox-auto-install-assistant` is the official Proxmox utility for preparing automated install
media. The script provides installation instructions if it's missing.

</div>
</div>

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Double-check which device is your USB before running the script. `lsblk` will show all block
devices and their sizes. The script prompts for confirmation, but it's worth verifying you're not
about to wipe the wrong drive. I almost nuked an external backup drive once. Same size as the USB. Wrong `/dev/sdX`. The confirmation prompt is there for a reason. Pause. Breathe. Verify.

</div>
</div>

---

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Test the full workflow in a VM before burning a USB for physical hardware. Create a VM in an
existing Proxmox host (or VirtualBox), attach the prepared ISO as a CD-ROM, and boot it. The
install runs identically in a VM. This lets you catch answer file typos and first-boot script
errors without walking to a server room.

</div>
</div>

---

## End-to-End Workflow

1. Clone the [repo](https://github.com/YOUR-USERNAME/proxmox-automated-install)
2. `cp answer.toml.example answer.toml`
3. Generate a root password hash: `mkpasswd --method=sha-512`
4. Fill in your network, disk, and credential details
5. `sudo ./prepare-usb.sh <iso> <usb>`
6. Boot from USB (check BIOS for the boot menu key)
7. Select **Automated Installation** (auto-selects after 10s)
8. Wait for install + reboot
9. First-boot script runs and configures everything
10. Hit the Web UI at `https://<MGMT_IP>:8006` and verify

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

The browser will warn about a self-signed certificate. This is expected. Accept the warning to
proceed. Log in as `root` with the password you hashed in `answer.toml`.

</div>
</div>

---

## What's Next

This gives you a fully configured Proxmox host from a single USB stick. Drive failure? Same
USB, same result. No runbooks, no tribal knowledge.

For ongoing management (backups, patching, security auditing), Ansible picks up where
`first-boot.sh` leaves off. The script logs a next-steps reminder at the end of its run.

---

## Related

- [Managing Proxmox with Ansible](/2026-02-09-ansible-proxmox-ops/) - Patching, backups, security after first-boot
- [Intel e1000e NIC fix](/2026-02-09-proxmox-intel-e1000e-fix/) - If your Dell OptiPlex drops off the network
- [Proxmox Backup Server](/2026-02-08-proxmox-backup-server/) - Automated VM backups

## References

- [Source code for this post](https://github.com/YOUR-USERNAME/proxmox-automated-install)
- [Proxmox Automated Installation Docs](https://pve.proxmox.com/wiki/Automated_Installation)
- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [fail2ban Documentation](https://www.fail2ban.org/)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
