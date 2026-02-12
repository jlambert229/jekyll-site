---
title: "Monitor Ansible Runs with Uptime Kuma"
subtitle: "Alert if site.yml or the patch playbook fails. Use Uptime Kuma's heartbeat monitor so a missed run triggers a notification."
date: 2026-02-09
tags:
  - ansible
  - monitoring
  - uptime-kuma
  - alerting
  - automation
categories:
  - Homelab Infrastructure
readtime: true
---
Scheduled Ansible runs fail silently. A host is unreachable, a variable is wrong, or SSH keys expired. You find out when the host drifts. Uptime Kuma's push monitor fixes that.
<!--more-->

> *"If you don't monitor it, you don't care about it."*

*"I've seen things you people wouldn't believe."* - Blade Runner. Your monitors have too. Ships on fire, pods OOMKilled, backups that never ran. Uptime Kuma tells you before you find out the hard way.

---

## The problem

Cron runs `ansible-playbook playbooks/site.yml` weekly. It fails. Maybe:
- SSH key expired
- Host was rebooted during the run
- Variable typo in the last commit
- NAS (NFS) was unreachable

The cron job exits non-zero. Nothing notifies you. Next week you realize the hosts haven't been patched in a month. I discovered my patch cron had been failing for six weeks when I SSHed into a node to debug something else and `uname -r` showed a kernel from two months ago. No alerts. No idea. The Intel e1000e fix? Never applied. That host had been dropping off the network randomly the whole time. Monitoring isn't optional.

> *"No alert is the worst alert. You want to know when things stop working - not when you eventually notice."* - Observability principle

---

## The solution: Push monitor

[Uptime Kuma](/2026-02-08-k8s-uptime-kuma/) has a **Push** (heartbeat) monitor type. Your script sends an HTTP request to a unique URL when it succeeds. If Uptime Kuma doesn't receive a push within the heartbeat interval, it alerts.

```
Cron runs site.yml → Success → curl HEARTBEAT_URL → Uptime Kuma records "up"
Cron runs site.yml → Fail    → (no curl)         → Uptime Kuma: no push in 36h → Alert
```

---

## Setup

### 1. Create a Push monitor in Uptime Kuma

1. Add Monitor → **Push**
2. Name: `Ansible Proxmox - site.yml`
3. Heartbeat interval: `36 hours` (or 2× your run frequency; if weekly, 36h gives one grace period)
4. Save. Copy the **push URL** (e.g. `https://uptime.media.lan/api/push/xxxxx`)

### 2. Add the heartbeat to your run script

```bash
#!/usr/bin/env bash
# Run site.yml and report success to Uptime Kuma
set -e

HEARTBEAT_URL="https://uptime.media.lan/api/push/xxxxxxxx"

cd /opt/proxmox-ops
if ansible-playbook playbooks/site.yml; then
  curl -fsS -m 10 "$HEARTBEAT_URL?status=up" || true
else
  # Don't push on failure - Uptime Kuma will alert when heartbeat is missed
  exit 1
fi
```

Store the URL in an env var or config file. Don't commit it to Git if the endpoint is sensitive (the push URL is a secret - anyone with it can fake "up" status).

### 3. Optional: Push failure details

You can append `&msg=` to the URL to pass a short message:

```bash
if ansible-playbook playbooks/site.yml; then
  curl -fsS -m 10 "$HEARTBEAT_URL?status=up" || true
else
  curl -fsS -m 10 "$HEARTBEAT_URL?status=down&msg=playbook_failed" || true
  exit 1
fi
```

Pushing "down" immediately can alert faster. Or omit it and let the missed heartbeat do the job (simpler).

---

## Heartbeat interval

| Run frequency | Heartbeat interval | Why |
|---------------|-------------------|-----|
| Weekly (Sat 3 AM) | 36 hours | One run per week; 36h allows for a missed weekend |
| Daily | 48 hours | Buffer for a missed day |
| Twice weekly | 5 days | ~2× run frequency |

Too short: false alerts when you skip a run. Too long: slow to detect real failures.

---

## Separate monitors per playbook

Create different push monitors for:
- **site.yml** - Full config (run weekly or monthly)
- **patch.yml** - Updates only (run weekly)

Each has its own URL. Each gets its own alert. Lets you distinguish "patching failed" from "full config failed."

---

## What I learned

> *"If it's not monitored, it will fail. If it's not alerted, you won't know. If you don't care, why are you running it?"* - The monitoring hierarchy

**The push URL is a secret.** I almost committed mine. "It's just a URL." Anyone with it can fake "up" status. Your Ansible could fail for a month and Uptime Kuma would never know. Keep it in an env var or a file outside the repo. Treat it like a password. Because it kind of is.

**Heartbeat > status page for cron.** A status page shows "last checked: 3 days ago." It does *not* alert you. I ran Ansible cron for two months before adding the heartbeat. Found out the hard way it had been failing for three weeks. Cause: the SSH key on the cron runner had expired (I'd rotated keys and forgot the runner). The playbook was failing on "Gathering Facts." I had no idea. Three weeks of unpatched Proxmox hosts. Push monitoring exists because cron fails silently. Use it. Your future self will sleep better.

**Match interval to run frequency.** Site runs weekly? Heartbeat at 36–48h. I set mine at 24h initially. Got an alert because I skipped a run (vacation). False positive. Annoying. Too short = false alerts. Too long = you don't notice real failures for days. Gives one or two grace periods. Tune it.

---

## References

- [Uptime Kuma Push monitor](https://github.com/louislam/uptime-kuma/wiki/Push-Based-Monitoring)
- [Uptime Kuma on Kubernetes](/2026-02-08-k8s-uptime-kuma/)
- [Scheduled Ansible runs](/2026-02-09-ansible-scheduled-runs/)
- [Managing Proxmox with Ansible](/2026-02-09-ansible-proxmox-ops/)
