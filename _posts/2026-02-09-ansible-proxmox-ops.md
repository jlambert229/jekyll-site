---
title: "Managing Proxmox Hosts with Ansible"
subtitle: "First-boot configures the host. Ansible keeps it that way. Patching, security hardening, and automated backups - run it once or schedule it."
date: 2026-02-09
tags:
  - ansible
  - proxmox
  - homelab
  - automation
  - patching
  - security
  - backup
categories:
  - Homelab Infrastructure
readtime: true
---
The [automated Proxmox install](/2026-02-07-proxmox-automated-install/) gets you a configured host from a USB stick. But what happens in month two? Package updates, SSH hardening drift, backup schedule changes. Manual changes on a single host become tribal knowledge. Add a second node and you're copy-pasting configs.

Ansible picks up where first-boot leaves off. *"The spice must flow."* - Dune. So must your config. Automate it.

<!--more-->

> *"It works on my machine"* - The other machine hasn't been updated in six months

![Works on my machine - Developer vs PM](/assets/img/memes/works-on-my-machine.png)

---

## Why Ansible for Proxmox?

I ran three Proxmox hosts (Dell OptiPlex 7080, plus two older boxes). Each had drifted:
- `fail2ban` - different `bantime` on each, one had `maxretry` at 3 (locking me out constantly)
- UFW - one host forgot `allow 3128` so vzdump-to-PBS failed silently; another had `allow 8006` from the wrong subnet
- Backup retention - one kept 90 days (ate 800 GB), the others 30; nobody could remember which was "correct"
- Package versions - one missed `6.8.12-9-pve`, the kernel that fixed the [Intel e1000e NIC hang](/2026-02-09-proxmox-intel-e1000e-fix/). That host kept dropping off the network during backups.

Reconciling them took a weekend. I should've been running Ansible from day one.

> *"If a human operator needs to touch your system during normal operations, you have a bug."* - John Allspaw, former VP Eng at Etsy

**Ansible gives you:**
- **Declarative config** - State in YAML, not scattered shell history
- **Idempotent** - Run it repeatedly, same outcome (*"What we do in life echoes in eternity."* - Gladiator. What Ansible does, it does the same way every time.)
- **Targeted** - Patch one host, harden all, or run the full stack
- **Auditable** - Git history shows when and why things changed

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  proxmox-ops (Ansible repo)                                 │
│                                                              │
│  playbooks/                                                  │
│   ├── site.yml      → All roles (patching, security, backup)│
│   ├── patch.yml     → Updates only                           │
│   ├── security.yml  → SSH, UFW, fail2ban, sysctl            │
│   └── backup.yml    → vzdump config, retention, schedules    │
│                                                              │
│  roles/                                                      │
│   ├── patching      → Repos, apt, pre/post checks, reboot   │
│   ├── security      → fail2ban, firewall, SSH hardening      │
│   └── backup        → vzdump.conf, cron, prune scripts      │
│                                                              │
│  inventory/                                                  │
│   ├── hosts.yml     → opti-7080, pve-node-2, ...             │
│   └── group_vars/   → Shared vars for proxmox_hosts         │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Start

```bash
# Clone and install collections
git clone https://github.com/YOUR-USERNAME/proxmox-ops.git
cd proxmox-ops
ansible-galaxy collection install -r requirements.yml

# Verify connectivity
aping proxmox_hosts
# or: ansible proxmox_hosts -m ping

# Full configuration (patching + security + backup)
ap playbooks/site.yml

# Dry run first? Use check mode
acheck playbooks/site.yml
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

With Ansible zsh aliases: `aping` pings hosts, `ap` runs playbooks, `acheck` runs check mode, `ahosts` lists inventory. Or use raw `ansible` / `ansible-playbook` commands.

</div>
</div>

---

## Roles in Detail

### Patching

Keeps repos and packages current. Proxmox ships with enterprise repos enabled - without a subscription, `apt update` fails. The patching role switches to no-subscription and manages updates.

**Pre-patch checks:** Disk space, no active migrations, services healthy. Skips if something's wrong.

**Post-patch checks:** Verifies services restarted, reports what changed. Handles reboot policy (manual, auto, or scheduled).

> *"Toil is the kind of work that scales linearly as a service grows. Automate it early."* - Site Reliability Engineering

```bash
# See what would be updated (no changes)
ap playbooks/patch.yml

# Apply updates (prompts for confirmation)
ap playbooks/patch.yml -e manual_patch=true

# Apply and auto-reboot if kernel update
ap playbooks/patch.yml -e manual_patch=true -e patching_reboot_policy=auto
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Reboot during migrations</div>
<div class="admonition-content" markdown="1">

If you're live-migrating VMs between nodes, don't run patching with `patching_reboot_policy=auto` on all hosts at once. Patch one node, migrate VMs away, reboot, then patch the next.

</div>
</div>

### Security

SSH hardening, UFW firewall, fail2ban for brute-force protection, sysctl tuning. Each control is a discrete task - enable or disable via variables.

| Control | Default | Description |
|---------|---------|-------------|
| SSH | Key-only, modern ciphers | Disables password auth, root login via key |
| UFW | Default deny | Allowlist for SSH, Web UI, SPICE, VNC |
| fail2ban | SSH + Proxmox Web UI | 5 failures in 10 min → 1 hour ban |
| sysctl | Spoofing, redirects | Kernel hardening for exposed hosts |

```bash
# Security hardening only
ap playbooks/security.yml

# Single host (e.g., before adding to cluster)
ap playbooks/security.yml --limit opti-7080
```

<div class="admonition admonition-production" markdown="1">
<div class="admonition-title">🏭 In production</div>
<div class="admonition-content" markdown="1">

For production, scope UFW rules to management VLANs (source IP allowlist). Consider Proxmox's built-in firewall (pve-firewall) for per-VM rules. See [Proxmox firewall docs](https://pve.proxmox.com/wiki/Firewall).

</div>
</div>

### Backup

Configures vzdump backups with cron, retention pruning, and integrity verification. The role writes `/etc/vzdump.conf`, drops cron scripts, and sets up a status reporter.

```bash
# Backup configuration only
ap playbooks/backup.yml
```

**Key variables** (in `inventory/group_vars/proxmox_hosts.yml`):

| Variable | Default | Description |
|----------|---------|-------------|
| `backup_schedule` | `0 2 * * *` | Cron: 2 AM daily |
| `backup_storage` | `nfs-synology` | Proxmox storage target |
| `backup_retention.keep_daily` | `7` | Days of daily backups |
| `backup_retention.keep_weekly` | `4` | Weekly backups |
| `backup_retention.keep_monthly` | `12` | Monthly backups |

---

## Adding a New Proxmox Host

**Automated** (recommended):

```bash
./scripts/add-proxmox-host.sh 192.168.2.132 opti-7090
```

The script discovers the host (or uses the IP you provide), adds it to inventory with the same config as existing hosts, and runs `site.yml`. New host gets patching, security, and backups in one pass.

**Prerequisite:** The new host must have your SSH key. Use [proxmox-automated-install](/2026-02-07-proxmox-automated-install/) first-boot or manual setup.

**Manual** (if you prefer):

```yaml
# inventory/hosts.yml
all:
  children:
    proxmox_hosts:
      hosts:
        opti-7090:
          ansible_host: 192.168.2.132
          ansible_user: jlambert
          ansible_become: true
          ansible_python_interpreter: /usr/bin/python3
          pve_node_name: opti-7090
          pve_management_ip: 192.168.2.132
```

Then run `ap playbooks/site.yml` or `ap playbooks/site.yml --limit opti-7090`.

---

## Tagged Runs

Run specific roles without touching the rest:

```bash
# Patching only
ap playbooks/site.yml --tags patching

# Security only
ap playbooks/site.yml --tags security

# Backup only
ap playbooks/site.yml --tags backup

# Skip security (e.g., debugging firewall)
ap playbooks/site.yml --skip-tags security
```

---

## Linting and Verification

```bash
# Lint playbooks and roles
alint

# Or explicitly
ansible-lint playbooks/ roles/
```

Catches common issues: undefined variables, deprecated patterns, risky tasks. Run before committing.

> *"Hope is not a strategy. If you're hoping it works, you haven't verified it."* - Every SRE who learned the hard way

---

## What I Learned

### 1. Run Site Weekly, Not Ad-Hoc

> *"Chaos is the absence of routine. Routine is the absence of chaos."* - Homelab operator's paradox

I used to run Ansible only when something broke. Then I spent a Sunday discovering that *three* Proxmox hosts had silently drifted - different UFW rules, different fail2ban configs, one running a kernel from six months ago. The NIC driver bug fix? Never applied. I looked at my calendar and realized I hadn't run the playbook in 47 days. I am not proud. Now I run `site.yml` weekly. Takes 5 minutes. Catches drift before it becomes a 4-hour reconciliation session.

### 2. Check Mode First, Always

New role? Changed variables? Run `acheck playbooks/site.yml` first. I didn't once. The playbook said "3 to change, 1 to destroy." I assumed it meant something harmless. It meant the backup retention role was about to *prune 60 days of backups* because I'd typo'd a variable. I caught it in the output at the last second. I still have nightmares. Firewall mistakes lock you out. Retention mistakes delete your safety net. Check mode is free.

### 3. One Inventory, Many Use Cases

Same `proxmox_hosts` group for site, patch, security, backup. Use `--limit` and `--tags` to narrow scope. I used to maintain separate inventories for "patch runs" vs "full config" and they drifted out of sync. Took me an embarrassing hour to figure out why `opti-7090` wasn't in the patch inventory. Spoiler: I'd added it to the wrong file. One inventory. One source of truth. Don't be me.

### 4. Fail2ban Can Lock You Out

First time I deployed fail2ban, I triggered it testing SSH. From my own IP. Banned myself. At 11pm. The Proxmox hosts were headless. I had to drive to the datacenter... wait, no, it's a homelab. I had to walk to the other room, plug in a monitor and keyboard, and console in like it was 1998. `fail2ban-client set sshd unbanip <my-ip>`. My dignity never recovered. Now I add my IP to `ignoreip` in `group_vars` *before* the first run. Or test from my phone's hotspot. Learn from my shame.

### 5. Backup Retention Is a Policy Decision

Started with 90-day daily retention. Hit 2 TB on the NAS in six weeks. My spouse asked why the backup share was bigger than our movie library. I had no good answer. Dropped to 7 daily, 4 weekly, 12 monthly. Still have 6 months of weeklies and a year of monthlies. The NAS stopped sending passive-aggressive disk-space alerts. Tune for your storage and your household's tolerance for "why is that folder 2 terabytes."

---

## What's Next

You have declarative config for Proxmox hosts. Patching, security, and backups in version control. Add a host by running one script.

**Optional enhancements:**
- [Vault for secrets](/2026-02-09-ansible-vault-secrets/) - Encrypt API keys, backup passwords in `group_vars/secrets.yml`
- [CI/CD with GitHub Actions](/2026-02-09-ansible-cicd-github-actions/) - `ansible-lint` and syntax check on every PR
- [Scheduled runs](/2026-02-09-ansible-scheduled-runs/) - Cron `patch.yml` weekly
- [Monitoring with Uptime Kuma](/2026-02-09-ansible-monitoring-uptime-kuma/) - Alert if playbooks fail (Push/heartbeat monitor)

First-boot gets the host online. Ansible keeps it consistent. Same result every time.

---

## References

- [proxmox-ops on GitHub](https://github.com/YOUR-USERNAME/proxmox-ops)
- [Automated Proxmox Install](/2026-02-07-proxmox-automated-install/) - First-boot setup
- [Ansible best practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Proxmox vzdump](https://pve.proxmox.com/pve-docs/vzdump.1.html) - Backup format and options
