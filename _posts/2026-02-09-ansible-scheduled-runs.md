---
title: "Scheduled Ansible Runs with Cron"
subtitle: "Run the patch playbook weekly. Keep Proxmox hosts updated without remembering to do it."
date: 2026-02-09
tags:
  - ansible
  - cron
  - automation
  - patching
categories:
  - Homelab Infrastructure
readtime: true
---
Ansible works when you run it. Ad-hoc runs drift. Schedule the patch playbook weekly and forget about it.
<!--more-->

> *"Automation that doesn't run is just documentation."*

*"Same time tomorrow?"* - Groundhog Day. Your cron should run the same playbook, same time, every week. Repetition is the point.

---

## Why schedule?

I used to run `apt update && apt upgrade` when something broke. Or when I remembered. Proxmox hosts drifted - one had a kernel from three months ago. The NIC bug fix? Never applied.

**Scheduled runs:** Same time every week. No human in the loop. Drift gets caught before it becomes a problem.

> *"Manual operations don't scale. The only way to go fast is to automate the boring stuff."* - DevOps Handbook

---

## The cron job

```bash
# /etc/cron.d/ansible-proxmox-patch
# Run patch playbook weekly (Saturdays at 3 AM)
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=admin@example.com

0 3 * * 6 ansible-runner run /opt/ansible-proxmox-patch 2>&1 | logger -t ansible-patch
```

Or with a wrapper script (recommended):

```bash
#!/bin/bash
# /usr/local/bin/proxmox-patch-cron.sh
set -e
cd /opt/proxmox-ops  # or wherever the repo lives
/usr/bin/ansible-playbook playbooks/patch.yml -e manual_patch=true \
  >> /var/log/ansible-patch.log 2>&1
```

Cron:

```cron
# Run Saturdays at 3 AM
0 3 * * 6 /usr/local/bin/proxmox-patch-cron.sh
```

---

## Key flags

| Flag | Purpose |
|------|---------|
| `-e manual_patch=true` | Applies updates (otherwise patch playbook only reports) |
| `--limit opti-7080` | Patch one host at a time (safer for clusters) |
| `-e patching_reboot_policy=auto` | Reboot if kernel update (use with caution) |

Default `patching_reboot_policy: manual` means the playbook applies packages but doesn't reboot. You reboot during a maintenance window.

---

## Where to run

**Options:**

1. **Dedicated workstation** - Always-on machine with SSH keys to Proxmox. Cron runs locally.
2. **Proxmox host** - One node runs cron and patches the others (and itself last). Requires SSH keys between nodes.
3. **GitHub Actions** - Scheduled workflow. Needs self-hosted runner or network access to hosts; otherwise use a different trigger.

For homelab, a NAS or always-on Linux box works. The cron user needs:
- Ansible installed
- SSH key to `ansible_user` on each Proxmox host
- Repo checked out (or use `ansible-pull`)

---

## Vault password

If you use [Ansible Vault](/2026-02-09-ansible-vault-secrets/), the cron job needs the password:

```bash
export ANSIBLE_VAULT_PASSWORD_FILE=/etc/ansible/.vault-pass
/usr/bin/ansible-playbook playbooks/patch.yml -e manual_patch=true
```

Store `/etc/ansible/.vault-pass` with restricted permissions (`chmod 600`). Or use `--ask-vault-pass` and an expect script - messier.

---

## Logging and alerting

Append to a log file:

```bash
/usr/bin/ansible-playbook playbooks/patch.yml -e manual_patch=true \
  >> /var/log/ansible-patch.log 2>&1
```

Pipe to `logger` for syslog:

```bash
... 2>&1 | logger -t ansible-patch
```

For alerts on failure, add exit-code handling:

```bash
/usr/bin/ansible-playbook playbooks/patch.yml -e manual_patch=true
STATUS=$?
if [ $STATUS -ne 0 ]; then
  echo "Ansible patch failed with $STATUS" | mail -s "Proxmox patch failed" admin@example.com
fi
exit $STATUS
```

Or integrate with [monitoring](/2026-02-09-ansible-monitoring-uptime-kuma/) (heartbeat on success, alert on missed heartbeat).

---

## site.yml vs patch.yml

| Playbook | Schedule | Purpose |
|----------|----------|---------|
| `patch.yml` | Weekly | Updates only. Fast. Low risk. |
| `site.yml` | Monthly or on change | Full config. Patching + security + backup. Catches drift. |

Run `patch.yml` weekly. Run `site.yml` when you change playbooks or add hosts. Or schedule `site.yml` monthly as a full reconciliation.

---

## What I learned

> *"Automation without monitoring is blind. Add a heartbeat so you know when it stops beating."* - See [monitoring](/2026-02-09-ansible-monitoring-uptime-kuma/)

**Start with patch, not site.** I scheduled `site.yml` weekly. Full config every time. Overkill. Patching is the high-frequency need - packages drift constantly. Security and backup config change maybe once a quarter. Run patch weekly. Run site monthly. Or when you change things. Your cron will thank you.

**Don't auto-reboot in cron.** I made this mistake once. `patching_reboot_policy: auto`. Saturday 3am. Cron ran. Kernel update on `pve-node-2`. Host rebooted. *While I had a VM live-migrating off it* - I'd started the migration before bed. The migration failed mid-transfer. The VM (Plex, naturally) went down. I woke up to Uptime Kuma alerts and an angry household. Learn from my terrible life choices. Reboot during a maintenance window. When you're awake. With coffee.

**Log somewhere.** Cron ran. Something failed. I had no idea what. Or when. Or why. The only evidence was "Ansible patch didn't run this week" from the monitoring alert. Log to `/var/log/ansible-patch.log` or syslog. When something breaks at 3am, you need to know what actually ran. Trust me on this.

---

## References

- [Managing Proxmox with Ansible](/2026-02-09-ansible-proxmox-ops/)
- [Patching role](https://github.com/YOUR-USERNAME/proxmox-ops) - `patching_reboot_policy`, `manual_patch`
