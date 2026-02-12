---
title: "Automated Proxmox Backups with Proxmox Backup Server"
subtitle: "Set up Proxmox Backup Server on your NAS, automate VM/CT backups, and sleep better knowing your homelab is recoverable."
date: 2026-02-08
tags:
  - proxmox
  - backup
  - disaster-recovery
  - homelab
  - automation
  - synology
categories:
  - Homelab Infrastructure
readtime: true
---
Proxmox Backup Server (PBS) running on your Synology NAS. Automated VM and container backups with deduplication, compression, and incremental snapshots. Disaster recovery for your homelab in one script.
<!--more-->

> *"I should probably back up my VMs... tomorrow"* - Famous last words before hardware failure

![This is fine - Before you had backups](/assets/img/memes/this-is-fine.jpeg)

---

## The Problem I Had

Ran a Proxmox homelab for two years. No backups. "I'll set that up next month." One thunderstorm - power flickered. The UPS held for 30 seconds. The primary NVMe (Samsung 970 EVO in the OptiPlex) didn't survive the brownout. SMART showed reallocated sectors. Then the drive stopped responding entirely. Lost:
- 3 Talos Kubernetes nodes (control plane + 2 workers)
- Pi-hole VM
- Dev environment with months of local Terraform state
- Grafana dashboards and Prometheus config

Spent a weekend rebuilding from memory. Some configs - custom UFW rules, specific fail2ban whitelists - were lost forever. The Homelab Tax, paid in full.

> *"The cost of a backup is always less than the cost of recovery without one."* - Disaster recovery math

*"What's the worst that could happen?"* - Famous last words. Then the SSD died. With PBS, your VMs can come back. Learned the hard way: **backups are not optional**.

---

## Why Proxmox Backup Server

Proxmox Backup Server is purpose-built for Proxmox environments:
- **Incremental backups** - First backup is full, subsequent ones are diffs (fast + space-efficient)
- **Deduplication** - Multiple VMs with similar data share blocks (Linux base images backed up once)
- **Compression** - zstd compression (fast + high ratio)
- **Verification** - Automated integrity checks
- **Web UI** - Browse backups, restore VMs with a few clicks
- **Scheduled jobs** - Cron-style automation (daily, weekly)

Alternative: `vzdump` to NFS share. Works, but no deduplication, no verification, manual retention management. PBS is better for multi-VM environments.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Proxmox VE (192.168.2.10)                                   │
│       │                                                      │
│   VMs: Talos K8s (CP + 2 workers), Pi-hole, dev box        │
│       │                                                      │
│       ▼                                                      │
│  Backup Jobs (daily 2am)                                    │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────┐                                │
│  │ Proxmox Backup Server   │                                │
│  │ (Synology NAS Docker)   │                                │
│  │ 192.168.2.10:8007      │                                │
│  └──────────┬──────────────┘                                │
│             │                                                │
│       ┌─────┴─────┐                                          │
│       │  Datastore │                                         │
│       │  /volume1/backups/proxmox  (deduplicated)           │
│       └───────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

PBS runs in Docker on your Synology (repurposing existing hardware). Proxmox connects to it, backs up VMs/CTs automatically.

---

## Prerequisites

- Synology NAS with Docker support
- SSH access to Synology (`jlambert@192.168.2.10`)
- Proxmox VE 8.x
- Static IP for PBS (or reserve DHCP lease)

---

## Deployment

Full source: [proxmox-backup-server-docker on GitHub](https://github.com/YOUR-USERNAME/proxmox-backup-server-docker)

Repo structure:
```
proxmox-backup-server-docker/
├── docker-compose.yml       # PBS container definition
├── .env.example             # PBS admin password
├── deploy.sh                # Automated deployment from workstation
├── verify.sh                # Health check script
└── README.md
```

---

## Docker Compose Configuration

PBS runs as a privileged container (needs direct disk access for deduplication):

```yaml
services:
  proxmox-backup-server:
    container_name: proxmox-backup-server
    image: proxmox-backup-server:latest
    restart: unless-stopped
    hostname: pbs
    privileged: true
    network_mode: host
    volumes:
      - /volume1/docker/pbs/etc:/etc/proxmox-backup
      - /volume1/docker/pbs/lib:/var/lib/proxmox-backup
      - /volume1/backups/proxmox:/mnt/datastore
    environment:
      - PBS_PASSWORD=${PBS_PASSWORD:-changeme}
      - PBS_EMAIL=admin@pbs.local
    ports:
      - "8007:8007"
```

**Key configuration:**

- **`privileged: true`** - PBS needs raw disk access for dedupe chunks
- **`network_mode: host`** - Simplifies Proxmox → PBS connectivity (no NAT translation)
- **Three volumes:**
  - `/etc` - PBS config (users, datastore definitions)
  - `/var/lib` - Internal state (chunk indexes, verification logs)
  - `/mnt/datastore` - Actual backup data (can be on separate filesystem/NFS mount)

Create `.env` file:

```bash
PBS_PASSWORD=your-secure-admin-password
```

---

## Deploy PBS on Synology

### Option 1: Automated Deployment

The repo includes `deploy.sh` - runs from your workstation over SSH:

```bash
git clone https://github.com/YOUR-USERNAME/proxmox-backup-server-docker.git
cd proxmox-backup-server-docker
./deploy.sh
```

What it does:
1. SSH to Synology
2. Creates directories (`/volume1/docker/pbs`, `/volume1/backups/proxmox`)
3. Prompts for PBS admin password
4. Copies `docker-compose.yml` and `.env`
5. Pulls image and starts container
6. Shows access URL and credentials

### Option 2: Manual Deployment

SSH to your Synology:

```bash
ssh jlambert@192.168.2.10

# Create directories
sudo mkdir -p /volume1/docker/pbs/{etc,lib}
sudo mkdir -p /volume1/backups/proxmox
sudo chown -R $(id -u):$(id -g) /volume1/docker/pbs /volume1/backups/proxmox

# Copy docker-compose.yml and .env
cd /volume1/docker/pbs
# (transfer files via scp or vi)

# Start PBS
docker-compose up -d

# Check logs
docker logs proxmox-backup-server
```

### Verify PBS is Running

```bash
./verify.sh
```

Or manually:

```bash
curl -k https://192.168.2.10:8007
# Should return: "API service ready"

docker ps | grep proxmox-backup-server
# Should show: Up <time>
```

Open PBS web UI: `https://192.168.2.10:8007`

- Username: `admin`
- Realm: `pbs`
- Password: (from `.env`)

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

PBS uses a self-signed certificate by default. Your browser will show a security warning. This is expected for homelab use. For production, configure a proper TLS cert.

</div>
</div>

---

## Configure PBS

### 1. Create Datastore

**Datastore → Add Datastore**

```yaml
Name: proxmox-vms
Path: /mnt/datastore
GC Schedule: daily at 03:00  # Garbage collection
Prune Schedule: daily at 04:00
Verify Schedule: weekly on Sunday at 05:00
```

**What these do:**

- **Garbage Collection (GC):** Removes orphaned chunks from deleted backups
- **Prune:** Applies retention policy (keep last N backups)
- **Verify:** Checksums all chunks to detect corruption

### 2. Set Retention Policy

**Datastore → proxmox-vms → Prune & GC → Edit Retention**

Recommended for homelabs:

```yaml
Keep Last: 7          # Last 7 backups (daily = 1 week)
Keep Hourly: 0
Keep Daily: 7         # 1 per day for 7 days
Keep Weekly: 4        # 1 per week for 4 weeks
Keep Monthly: 6       # 1 per month for 6 months
Keep Yearly: 2        # 1 per year for 2 years
```

This gives you:
- Granular recovery (7 days of daily backups)
- Long-term history (6 months + 2 years)
- Automatic cleanup (old backups pruned)

Example: After 8 days, the oldest daily backup is pruned but a weekly backup is kept.

### 3. Create Backup User (Optional but Recommended)

Don't use the `admin` account from Proxmox.

**Configuration → User Management → Add User**

```yaml
Username: proxmox-backup
Realm: pbs
Email: proxmox@yourdomain.com
```

**Permissions → Add → User Permission**

```yaml
Path: /datastore/proxmox-vms
User: proxmox-backup@pbs
Role: DatastoreBackup
```

Generate API token:

**Configuration → Access Control → API Tokens → Add**

```yaml
User: proxmox-backup@pbs
Token ID: backup-token
Privilege Separation: Yes
```

Copy the token (shown once). You'll need it for Proxmox configuration.

---

## Configure Proxmox to Use PBS

### 1. Add PBS as Storage

Proxmox Web UI → Datacenter → Storage → Add → Proxmox Backup Server

```yaml
ID: pbs-synology
Server: 192.168.2.10
Username: proxmox-backup@pbs
Password/Token: (paste API token or password)
Datastore: proxmox-vms
Fingerprint: (click "Scan" to auto-fill)
```

**Verify:**

- Storage list should show `pbs-synology` with green checkmark
- Click on it → Content → Should show "No backups" (initially)

### 2. Create Backup Job

**Datacenter → Backup → Add**

```yaml
Node: pve (or your node name)
Storage: pbs-synology
Schedule: Daily at 02:00
Selection Mode: All
Compression: zstd
Mode: Snapshot
Protected: No
Send email to: your-email@domain.com
```

**Mode explained:**

- **Snapshot:** VM keeps running, backup is a point-in-time snapshot (preferred)
- **Suspend:** VM paused during backup (ensures consistency for databases)
- **Stop:** VM stopped, backed up, restarted (longest downtime)

For most homelab VMs, **Snapshot** is fine. For critical databases, consider **Suspend**.

### 3. Run First Backup (Manual Test)

**Datacenter → Backup → Select job → Run now**

Watch the task log. First backup is full (slow). Subsequent backups are incremental (fast).

Check PBS Web UI → Datastore → proxmox-vms → Content

You should see backup entries for each VM.

---

## Scheduled Backups

With the backup job configured, Proxmox handles automation:

- **Daily 2am:** Full backup of all VMs (incremental after first run)
- **Daily 3am:** PBS runs garbage collection
- **Daily 4am:** PBS prunes old backups per retention policy
- **Weekly Sunday 5am:** PBS verifies chunk integrity

Zero manual intervention. Backups just happen.

---

## Restore VMs

### Restore Entire VM

**Proxmox UI → Storage → pbs-synology → Content**

1. Select a VM backup
2. Click "Restore"
3. Choose VM ID (existing or new)
4. Click "Restore"

Proxmox downloads from PBS and creates the VM. Takes 5-15 minutes depending on VM size.

### Restore Single Disk

If only a disk is corrupted:

1. Storage → pbs-synology → Content → Select backup
2. Click "Show Configuration"
3. Select disk → "Restore"
4. Choose target VM and disk

### Restore to Different Proxmox Host (Disaster Recovery)

**Scenario:** Proxmox host died, you rebuilt on new hardware.

1. Install Proxmox on new host
2. Add PBS storage: Datacenter → Storage → Add → Proxmox Backup Server (same config as before)
3. Storage → pbs-synology → Content → Select backups → Restore

PBS holds the backups. Proxmox hosts are disposable.

---

## Monitoring and Alerts

### Email Notifications

PBS sends emails on backup completion/failure.

**Configuration → Administration → Email**

```yaml
SMTP Server: smtp.gmail.com
Port: 587
Username: your-email@gmail.com
Password: (app-specific password)
From: pbs@yourdomain.com
To: alerts@yourdomain.com
```

Test: **Configuration → Email → Send Test Email**

### Syslog Forwarding

Forward PBS logs to a central log server (optional):

**Configuration → Administration → Syslog**

```yaml
Server: 192.168.2.100:514
Protocol: UDP
Format: RFC3164
```

### Verify Job Status

**Dashboard** shows:

- Last backup time
- Success/failure count
- Datastore usage
- Upcoming scheduled tasks

Check weekly. If backups fail repeatedly, investigate.

---

## Backup Verification

PBS automatically verifies backups, but you should test restores manually.

**Quarterly disaster recovery drill:**

1. Pick a random VM backup
2. Restore to a test VM ID
3. Boot the VM
4. Verify functionality
5. Delete test VM

**Why:** Backups you haven't tested are Schrödinger's backups - simultaneously valid and useless.

---

## Troubleshooting

### Proxmox Can't Connect to PBS

**Symptom:** "Connection refused" or "Connection timed out"

**Check:**

1. **PBS is running:**
   ```bash
   ssh jlambert@192.168.2.10
   docker ps | grep proxmox-backup-server
   ```

2. **Firewall:**
   ```bash
   # On Synology
   sudo iptables -L | grep 8007
   # Should allow port 8007
   ```

3. **Network:**
   ```bash
   # From Proxmox node
   curl -k https://192.168.2.10:8007
   # Should return: "API service ready"
   ```

### Backup Job Fails with "No Space Left"

**Symptom:** Backup succeeds initially, then starts failing

**Check datastore usage:**

PBS Web UI → Datastore → proxmox-vms

- Used: X GB
- Available: Y GB

**Solutions:**

1. **Prune more aggressively:**
   - Reduce retention (keep last 7 → 3)
   - Run manual prune: Datastore → Prune & GC → Prune Now

2. **Garbage collect:**
   - Old chunks may not be cleaned up yet
   - Datastore → Prune & GC → GC Now

3. **Expand storage:**
   - Add another volume to Synology
   - Or create a second datastore

### Backup Verification Fails

**Symptom:** Weekly verify job reports errors

**Check:**

```bash
ssh jlambert@192.168.2.10
docker logs proxmox-backup-server | grep -i verify
```

Common causes:

- **Disk corruption** - Run Synology's disk check (Storage Manager → HDD/SSD → Health Info → S.M.A.R.T. Test)
- **Interrupted backup** - A backup was stopped mid-process (manual stop or crash)
- **Bad chunks** - Rare but possible (disk failure, bit rot)

**Fix:**

1. Identify corrupted backup: PBS UI → Datastore → Content → Look for red "!" icons
2. Delete corrupted backup: Select → Remove
3. Re-run backup job: Proxmox → Backup → Run now

### Restore Hangs

**Symptom:** Restore starts but never completes

**Check:**

1. **Network speed:**
   ```bash
   # From Proxmox node
   iperf3 -c 192.168.2.10
   # Should show > 100 Mbps
   ```

   If slow, check:
   - NIC duplex settings (force 1000baseT full duplex)
   - Switch port errors

2. **PBS load:**
   - PBS Web UI → Dashboard → CPU/Memory
   - If maxed out, wait for GC/verify jobs to finish

3. **Task log:**
   - Proxmox → Node → Tasks → Select restore task → Show log
   - Look for errors

---

## Resource Usage

**PBS Docker container:**

- **CPU:** 5-10% during backup, <1% idle
- **Memory:** 500 MB - 1 GB
- **Storage:** 
  - PBS image + config: ~500 MB
  - Datastore: Depends on VM count/size

**Example (5 VMs, 200 GB total):**

| Backup # | Type | Size | Duration | Datastore Growth |
|----------|------|------|----------|------------------|
| 1 | Full | 200 GB | 60 min | +80 GB (compressed+dedup) |
| 2 | Incremental | 10 GB changed | 10 min | +2 GB |
| 7 | Incremental | 15 GB changed | 12 min | +3 GB |

After 7 daily backups: ~100 GB used (50% savings from dedup+compression).

---

## What I Learned

### 1. Test Restores, Not Just Backups

I ran PBS backups for six months. Never tested a restore. "It's backing up. What could go wrong?" Then a VM corrupted - ext4 errors, `kernel: EXT4-fs error`, wouldn't boot past initramfs. I went to restore. The backup was there. The restore *failed*. Corrupted backup. Root cause: the PBS datastore volume (`/volume1/backups/proxmox`) had run out of space three months earlier. PBS had been writing partial chunks. No alert. PBS doesn't fail loudly when the destination fills - it just keeps going. I lost that VM. Rebuilt from scratch. I lost that VM. I rebuilt it from scratch. It took a day. I now restore a random VM every quarter. Trust nothing. Verify everything.

### 2. Incremental Backups Are Fast

First backup: 90 minutes. 5 VMs, 250 GB. I thought "this is going to be unsustainable." Daily backups now: 5–10 minutes. PBS's incremental snapshots are black magic. Only changed blocks. I don't understand how it's that fast. I don't care. It works.

### 3. Deduplication Saves Serious Disk Space

I have 3 Kubernetes VMs with identical Talos base images. PBS stores the base *once*. 3x VMs = 1.2x storage. I did the math: without dedup, I'd need 600 GB. I'm using 220. The Synology is happier. So is my wallet. Deduplication is the homelab operator's best friend.

### 4. Retention Policies Prevent "I'll Clean This Up Later" Debt

I used to manually delete old backups. "When I remember." I never remembered. The datastore grew. And grew. One day I looked: 400 GB of backups, some from six months ago. I'd restored exactly zero of them. Set retention once. PBS prunes. I never think about it. "I'll clean this up later" is the siren song of technical debt. Don't answer it.

### 5. PBS on Synology Is Better Than Dedicated Hardware

I almost bought a used laptop to run PBS. "Dedicated backup appliance." Then I realized: the Synology already has RAID, already runs 24/7, already gets Hyper Backups. Adding another device = another power supply, another thing that can fail, another thing to maintain. PBS in Docker on the NAS. One less device. One less cable. One less excuse for my spouse to ask "how many computers do we have?"

---

## What's Next

You have automated Proxmox backups with deduplication, compression, and retention management. Disaster recovery is now measured in minutes, not days.

**Optional enhancements:**

- **Offsite replication** - Sync datastore to a remote PBS instance (rsync over SSH)
- **Encrypted backups** - PBS supports client-side encryption (Proxmox encrypts before sending)
- **S3 upload** - Push backups to Backblaze B2 for geographic redundancy
- **Monitoring integration** - Alert on failed backups via [Uptime Kuma](/2026-02-08-k8s-uptime-kuma/)

**Related:** [Velero for Kubernetes](/2026-02-08-k8s-velero-backups/) - Cluster-level backups; [Ansible Proxmox ops](/2026-02-09-ansible-proxmox-ops/) - Automate vzdump schedules.

The core setup is production-ready. Sleep better knowing your homelab is recoverable.

---

## References

- [Proxmox Backup Server Documentation](https://pbs.proxmox.com/docs/)
- [Proxmox VE Backup Guide](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
- [PBS Docker Image](https://hub.docker.com/r/proxmox/proxmox-backup-server)
- [PBS Forum](https://forum.proxmox.com/forums/proxmox-backup-server.130/)
