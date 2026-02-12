---
title: "Self-Hosted Password Manager with Vaultwarden on Kubernetes"
subtitle: "Bitwarden-compatible password manager running on your homelab. End-to-end encrypted, browser extensions, mobile apps, zero recurring fees."
date: 2026-02-08
tags:
  - kubernetes
  - homelab
  - security
  - vaultwarden
  - bitwarden
  - helm
  - passwords
categories:
  - Kubernetes Homelab
readtime: true
---
Self-hosted password management with Vaultwarden (Bitwarden-compatible server). Browser extensions, mobile apps, end-to-end encryption, and zero subscription fees. All your credentials stay on your NAS.
<!--more-->

> *"I use the same password for everything"* - You, before reading this

*"I am the one who knocks."* - Breaking Bad. When you self-host, you control the keys. No SaaS vendor holding your passwords hostage. Vaultwarden puts them on *your* NAS.

---

## Why Vaultwarden

Used 1Password for years. $40/year subscription. Great product, but my passwords lived on someone else's server. I already run a homelab with backups and monitoring - why pay for cloud storage I don't control?

Vaultwarden is a lightweight, Rust-based implementation of the Bitwarden server. Compatible with official Bitwarden clients (browser extensions, mobile apps, CLI). Runs on a single Kubernetes pod with minimal resources.

**You get:**
- **Browser extensions** - Chrome, Firefox, Safari, Edge
- **Mobile apps** - iOS, Android (official Bitwarden apps)
- **Desktop apps** - Windows, macOS, Linux
- **CLI** - Scripting and automation
- **TOTP 2FA** - Built-in authenticator (no need for Authy/Google Authenticator)
- **Secure notes** - Store SSH keys, API tokens, recovery codes
- **Self-hosted** - Your data, your NAS, your control

No feature limitations (unlike Bitwarden's free tier). No recurring fees. Same workflow as 1Password/Bitwarden.

> *"Passwords are the keys to everything. Put them somewhere you control, with backups you've tested."* - Security hygiene

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Security Considerations</div>
<div class="admonition-content" markdown="1">

This deployment uses HTTPS with Let's Encrypt certs and stores the vault database on encrypted NFS. **Do not** expose Vaultwarden directly to the internet without additional hardening (fail2ban, Cloudflare Access, VPN, etc.). See the Security section below.

</div>
</div>

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Clients (Browser, Mobile, Desktop, CLI)                    │
│       │                                                      │
│   https://vault.media.lan (Traefik + cert-manager)          │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────┐                                    │
│  │   Vaultwarden       │                                    │
│  │   (SQLite)          │                                    │
│  └──────────┬──────────┘                                    │
│             │                                                │
│       ┌─────┴─────┐                                          │
│       │ NFS (PVC) │                                          │
│       │ Synology  │                                          │
│       └───────────┘                                          │
│                                                              │
│  Backups: Daily to separate PVC + offsite sync              │
└─────────────────────────────────────────────────────────────┘
```

---

## Deployment Repo

Full source: [k8s-vaultwarden on GitHub](https://github.com/YOUR-USERNAME/k8s-vaultwarden)

```
k8s-vaultwarden/
├── values.yaml              # bjw-s/app-template Helm values
├── cert.yaml                # cert-manager Certificate (Let's Encrypt)
├── deploy.sh                # Automated deployment
├── backup.sh                # SQLite backup script
└── README.md
```

---

## Prerequisites

From previous posts, you need:
- [Kubernetes cluster](/2026-02-08-k8s-talos-proxmox-deploy/) (Talos on Proxmox)
- [Traefik ingress](/2026-02-08-k8s-homelab-infra/) with LoadBalancer IP
- [cert-manager](/2026-02-08-k8s-homelab-infra/) for TLS certificates
- DNS entry: `vault.media.lan → <TRAEFIK_IP>`

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Vaultwarden **requires HTTPS**. Bitwarden clients refuse to connect to HTTP endpoints (security policy). This guide uses cert-manager with self-signed CA or Let's Encrypt.

</div>
</div>

---

## Helm Values

Uses the `bjw-s/app-template` chart (same pattern as media stack):

```yaml
# values.yaml
controllers:
  vaultwarden:
    strategy: Recreate
    containers:
      app:
        image:
          repository: vaultwarden/server
          tag: 1.32.1
          pullPolicy: IfNotPresent
        env:
          TZ: "America/New_York"
          DOMAIN: "https://vault.media.lan"
          SIGNUPS_ALLOWED: "true"   # Disable after creating your account
          INVITATIONS_ALLOWED: "true"
          SHOW_PASSWORD_HINT: "false"
          LOG_LEVEL: "info"
          EXTENDED_LOGGING: "true"
          LOG_FILE: "/data/vaultwarden.log"
          # Admin panel (disable in production or set strong token)
          # ADMIN_TOKEN: "your-admin-token-here"
        probes:
          liveness:
            enabled: true
            custom: true
            spec:
              httpGet:
                path: /alive
                port: 80
              initialDelaySeconds: 30
              periodSeconds: 10
          readiness:
            enabled: true
            custom: true
            spec:
              httpGet:
                path: /alive
                port: 80
              initialDelaySeconds: 30
              periodSeconds: 10
        resources:
          requests:
            cpu: 50m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi

service:
  app:
    controller: vaultwarden
    ports:
      http:
        port: 80

ingress:
  app:
    enabled: true
    className: traefik
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"  # Or "selfsigned-ca"
    hosts:
      - host: vault.media.lan
        paths:
          - path: /
            pathType: Prefix
            service:
              identifier: app
              port: http
    tls:
      - secretName: vaultwarden-tls
        hosts:
          - vault.media.lan

persistence:
  data:
    type: persistentVolumeClaim
    accessMode: ReadWriteOnce
    size: 1Gi
    storageClass: nfs-appdata
    globalMounts:
      - path: /data
```

**Key decisions:**

- **`SIGNUPS_ALLOWED: "true"`** - Enable for initial account creation. Disable after setup.
- **`DOMAIN`** - Must match your ingress hostname (used for email links).
- **HTTPS required** - Bitwarden clients enforce TLS.
- **1 GB storage** - SQLite database + attachments. Plenty for personal use.
- **`strategy: Recreate`** - SQLite doesn't support concurrent writes.

---

## Deploy

### 1. Create Namespace

```bash
kubectl create namespace security
```

### 2. Deploy Vaultwarden

```bash
helm repo add bjw-s https://bjw-s-labs.github.io/helm-charts/
helm repo update

helm upgrade --install vaultwarden bjw-s/app-template \
    -n security -f values.yaml --wait
```

Or use the script:

```bash
git clone https://github.com/YOUR-USERNAME/k8s-vaultwarden.git
cd k8s-vaultwarden
./deploy.sh
```

### 3. Verify

```bash
kubectl get pods -n security
kubectl get ingress -n security
kubectl logs -n security -l app.kubernetes.io/name=vaultwarden
```

Open `https://vault.media.lan`. You should see the Vaultwarden login page.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

If using a self-signed cert, your browser will show a security warning. This is expected. Accept and proceed. For production, use Let's Encrypt with a real domain.

</div>
</div>

---

## Initial Setup

### 1. Create Your Account

1. Open `https://vault.media.lan`
2. Click "Create Account"
3. Enter email and master password
   - **This password encrypts your vault.** No one can recover it if lost.
   - Use a strong, memorable passphrase (diceware, 6+ words)
   - Write it down on paper (seriously)
4. Create account

### 2. Disable Public Signups

After creating your account, prevent others from signing up:

```bash
# Edit values.yaml
SIGNUPS_ALLOWED: "false"

# Redeploy
helm upgrade vaultwarden bjw-s/app-template -n security -f values.yaml
```

### 3. Configure Browser Extension

Install Bitwarden extension:
- Chrome: [Chrome Web Store](https://chrome.google.com/webstore/detail/bitwarden/nngceckbapebfimnlniiiahkandclblb)
- Firefox: [Firefox Add-ons](https://addons.mozilla.org/en-US/firefox/addon/bitwarden-password-manager/)

Configure:
1. Click extension icon → Settings (gear)
2. Server URL: `https://vault.media.lan`
3. Log in with your email and master password

### 4. Install Mobile Apps

- **iOS:** [App Store](https://apps.apple.com/app/bitwarden-password-manager/id1137397744)
- **Android:** [Google Play](https://play.google.com/store/apps/details?id=com.x8bit.bitwarden)

On first launch:
1. Tap "Self-hosted"
2. Server URL: `https://vault.media.lan`
3. Log in

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Mobile apps cache credentials locally. You can access passwords even when your homelab is offline.

</div>
</div>

---

## Import Existing Passwords

### From 1Password

1. 1Password → File → Export → CSV (All Items)
2. Vaultwarden web vault → Settings → Import Data
3. Select "1Password (csv)" format
4. Upload file
5. Delete CSV from disk (contains plaintext passwords)

### From Chrome/Firefox

1. Browser → Settings → Passwords → Export
2. Save as CSV
3. Vaultwarden → Settings → Import Data → "Chrome (csv)"
4. Upload and delete CSV

### From Bitwarden Cloud

If migrating from Bitwarden's hosted service:

1. Bitwarden cloud → Settings → Export Vault
2. Download JSON (encrypted format preferred)
3. Vaultwarden → Import Data → "Bitwarden (json)"
4. Delete export file

---

## Security Hardening

### 1. Enable 2FA for Your Account

Web vault → Settings → Two-step Login

**Authenticator app (TOTP):**
1. Click "Manage" next to "Authenticator App"
2. Scan QR code with Vaultwarden's built-in TOTP feature or Authy/Google Authenticator
3. Save recovery code (store in a secure note in Vaultwarden itself)

**U2F/WebAuthn (YubiKey):**

If you have a hardware key:
1. Settings → Two-step Login → FIDO2 WebAuthn
2. Insert YubiKey, click "Add"
3. Name it and save

### 2. Set Admin Token (Disable Anonymous Admin Access)

The admin panel (`/admin`) lets you manage users, view logs, and delete accounts. By default, it's accessible without authentication (intentionally, for initial setup).

**Secure it:**

Generate a strong token:

```bash
openssl rand -base64 48
```

Update `values.yaml`:

```yaml
env:
  ADMIN_TOKEN: "<your-generated-token>"
```

Redeploy:

```bash
helm upgrade vaultwarden bjw-s/app-template -n security -f values.yaml
```

Access admin panel: `https://vault.media.lan/admin` (requires token)

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

**OR** disable the admin panel entirely once your account is created:
```yaml
ADMIN_TOKEN: "disabled"
```

</div>
</div>

### 3. Disable Public Signups

Already covered above. Ensure:

```yaml
SIGNUPS_ALLOWED: "false"
INVITATIONS_ALLOWED: "true"  # Only you can invite others
```

### 4. Fail2ban for Brute Force Protection

If exposing to the internet, add fail2ban to block repeated failed logins.

This requires access to Vaultwarden logs. Mount logs to a sidecar or use Kubernetes audit logs with a tool like Falco.

**Simpler:** Don't expose directly to internet. Use Tailscale/WireGuard VPN or Cloudflare Access.

### 5. Database Encryption at Rest

Vaultwarden's database stores encrypted vault data (your passwords are encrypted client-side). But metadata (email addresses, timestamps) is plaintext in SQLite.

**Add encryption:**

1. Synology: Enable encryption on the NFS share (`nfs01`)
2. Or use LUKS-encrypted volumes on your NAS

### 6. Regular Backups (Critical)

**Losing this database = losing all passwords.**

See the Backup Strategy section below.

---

## Backup Strategy

Vaultwarden's database is a single SQLite file: `/data/db.sqlite3`.

The repo includes `backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

NAMESPACE="security"
POD=$(kubectl get pod -n "$NAMESPACE" -l app.kubernetes.io/name=vaultwarden -o jsonpath='{.items[0].metadata.name}')
BACKUP_DIR="./backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

echo "Backing up Vaultwarden database..."

# SQLite backup (proper .backup command, not just cp)
kubectl exec -n "$NAMESPACE" "$POD" -- sqlite3 /data/db.sqlite3 ".backup /tmp/vaultwarden-backup.db"
kubectl cp -n "$NAMESPACE" "$POD:/tmp/vaultwarden-backup.db" "$BACKUP_DIR/vaultwarden-${TIMESTAMP}.db"
kubectl exec -n "$NAMESPACE" "$POD" -- rm /tmp/vaultwarden-backup.db

# Also backup attachments
kubectl exec -n "$NAMESPACE" "$POD" -- tar czf /tmp/attachments.tar.gz /data/attachments 2>/dev/null || true
kubectl cp -n "$NAMESPACE" "$POD:/tmp/attachments.tar.gz" "$BACKUP_DIR/attachments-${TIMESTAMP}.tar.gz" || true
kubectl exec -n "$NAMESPACE" "$POD" -- rm /tmp/attachments.tar.gz 2>/dev/null || true

echo "✅ Backup saved:"
echo "   Database: $BACKUP_DIR/vaultwarden-${TIMESTAMP}.db"
echo "   Attachments: $BACKUP_DIR/attachments-${TIMESTAMP}.tar.gz"

# Keep last 30 backups
ls -t "$BACKUP_DIR"/vaultwarden-*.db | tail -n +31 | xargs -r rm
ls -t "$BACKUP_DIR"/attachments-*.tar.gz | tail -n +31 | xargs -r rm
```

Run manually:

```bash
./backup.sh
```

**Automate with cron:**

```bash
# Daily backup at 4am
0 4 * * * /path/to/k8s-vaultwarden/backup.sh 2>&1 | logger -t vaultwarden-backup
```

**Offsite sync:**

```bash
# Sync backups to Synology via rsync
rsync -avz ./backups/ jlambert@192.168.2.10:/volume1/backups/vaultwarden/
```

<div class="admonition admonition-danger" markdown="1">
<div class="admonition-title">🚨 Test Your Backups</div>
<div class="admonition-content" markdown="1">

Restore a backup to a test environment quarterly. Backups you haven't tested are Schrödinger's backups - simultaneously valid and useless.

</div>
</div>

---

## Usage Tips

### TOTP 2FA Codes in Vaultwarden

Store TOTP secrets directly in Vaultwarden (no need for separate authenticator app):

1. Edit a login item
2. Click "New Custom Field" → "TOTP"
3. Paste the secret key or scan QR code
4. Vaultwarden generates 6-digit codes automatically

Browser extension shows codes next to passwords. One less app to manage.

### Secure Notes for SSH Keys

Store SSH private keys, API tokens, and recovery codes:

1. New Item → Secure Note
2. Type: "Generic"
3. Paste SSH key or token
4. Optionally attach files (e.g., `.pem` files)

### Password Generator

Browser extension has a built-in generator:
- Length: 20-32 characters
- Include symbols, numbers, uppercase, lowercase
- Avoid ambiguous characters (O/0, l/1) for manual entry

### Emergency Access

Vaultwarden supports "Emergency Access" - designate a trusted contact who can request access to your vault after a waiting period.

Web vault → Settings → Emergency Access → Invite Trusted Emergency Contact

Use case: If you die, your spouse can access critical passwords after 30 days.

---

## Troubleshooting

### Browser Extension Won't Connect

**Symptom:** "Cannot connect to server"

**Check:**

1. **HTTPS required** - Bitwarden clients refuse HTTP
2. **DNS resolution** - Can you resolve `vault.media.lan`?
   ```bash
   nslookup vault.media.lan
   ```
3. **Certificate trust** - Self-signed certs need manual trust:
   - Chrome: `chrome://settings/certificates` → Import CA cert
   - Firefox: `about:preferences#privacy` → View Certificates → Import
4. **Traefik routing** - Verify ingress:
   ```bash
   kubectl get ingress -n security
   ```

### Mobile App Can't Connect

**Symptom:** "An error has occurred"

**Check:**

1. **Network** - Is your phone on the same LAN as the homelab?
2. **DNS** - Does your Pi-hole serve `vault.media.lan` to mobile devices?
3. **Certificate** - Self-signed certs may fail on iOS/Android. Use Let's Encrypt with a real domain or Tailscale for automatic cert trust.

### Forgot Master Password

**There is no recovery.** This is by design. Your vault is encrypted with your master password. No one (including Vaultwarden) can decrypt it without the password.

**Prevention:**

1. Write master password on paper, store in a safe
2. Use Emergency Access to designate a trusted contact
3. Back up your vault export periodically:
   - Web vault → Settings → Export Vault → JSON (encrypted)
   - Store export file offline

### Database Corruption

**Symptom:** Vaultwarden won't start, logs show "database disk image is malformed"

**Recovery:**

```bash
# 1. Scale down
kubectl scale -n security deploy/vaultwarden --replicas=0

# 2. Restore from backup
kubectl cp ./backups/vaultwarden-20260208_040000.db security/<pod>:/data/db.sqlite3

# 3. Scale up
kubectl scale -n security deploy/vaultwarden --replicas=1
```

**Prevention:** Regular backups. SQLite is resilient but not immune to corruption (power loss, disk failures).

---

## Resource Usage

Tested on 2-worker cluster (2 vCPU, 4 GB RAM per worker):

- **CPU:** <1% idle, <5% during sync
- **Memory:** 80-120 MB
- **Storage:** 50 MB (database + attachments for 500 logins)

Vaultwarden is remarkably efficient. One pod handles multiple users with ease.

---

## What I Learned

### 1. HTTPS Is Non-Negotiable

Tried HTTP initially. "It's internal, who cares?" The Bitwarden extension refused to connect. Security policy. I spent 20 minutes thinking the deployment was broken. The extension wouldn't even tell me why - just "connection failed." Turns out Bitwarden *requires* HTTPS. Even for self-hosted. Even on a local network. Set up cert-manager first. Don't be me.

### 2. Master Password Strategy Matters

I use a 6-word diceware passphrase. Memorable. High entropy. I wrote it down on paper and put it in a fire-safe. This is the one password you *cannot* reset. Lose it, lose everything. I know someone who forgot theirs - their Bitwarden master password, no recovery key. They had to use the export from an old backup (plaintext JSON), create a new vault, and re-add 400+ passwords manually. Banking, work SSO, crypto exchanges. Over a weekend. With a spreadsheet. It was not fun. Plan accordingly. Your future self will thank you.

### 3. TOTP in Vaultwarden Is a Game-Changer

I used Google Authenticator. Phone died. New phone. Authenticator codes... gone. Had to use backup codes for every account. Painful. Vaultwarden's built-in TOTP: passwords and 2FA in one place. One backup. One restore. I migrated everything. Never looked back. The convenience is unreal.

### 4. Offsite Backups Are Critical

Your password database is a single point of failure. House fire? Theft? Ransomware? I rsync backups to a Synology at a friend's house. Encrypted. Automated. If my house burns down, I still have access to everything. Morbid to think about. Essential to have. I set this up *after* I'd already lost a phone with Authenticator. Don't wait for disaster.

### 5. Emergency Access Saved Me Once

I forgot to renew my domain (the one I use for email - `me@myhomelab.com` or similar). It expired. Password resets go to that email. I couldn't receive them. I was locked out of my own vault. Couldn't log into Bitwarden. Couldn't get into work (SSO). Couldn't access banking. Full lockout. I had set up Emergency Access for my spouse months earlier - "just in case." The waiting period felt like forever. She granted access. I got in. I renewed the domain. I added a calendar reminder. That feature saved me from a very bad day. Set it up before you need it.

---

## What's Next

You have self-hosted password management running on your homelab. No subscription fees, full control, Bitwarden-compatible clients.

**Optional enhancements:**

- **Cloudflare Tunnel** - Expose Vaultwarden securely without port forwarding
- **Tailscale** - Access from anywhere via encrypted mesh VPN
- **Automated backups** - Integrate with [Velero](/2026-02-08-k8s-velero-backups/) for cluster-level backup
- **Multi-user** - Invite family members, share passwords via Organizations
- **Hardware key enforcement** - Require YubiKey for all logins

The core setup is production-ready. Migrate your passwords and delete your 1Password subscription.

---

## Related

- [Pi-hole on Synology](/2026-02-08-pihole-synology-docker/) - Same Docker-on-NAS pattern, different use case
- [K8s Homelab Infrastructure](/2026-02-08-k8s-homelab-infra/) - Prerequisites: Traefik, cert-manager, LoadBalancer
- [Velero Backups](/2026-02-08-k8s-velero-backups/) - Backup the PVC; password databases are critical

## References

- [Vaultwarden GitHub](https://github.com/dani-garcia/vaultwarden)
- [Vaultwarden Wiki](https://github.com/dani-garcia/vaultwarden/wiki)
- [Bitwarden Clients](https://bitwarden.com/download/)
- [bjw-s app-template docs](https://bjw-s-labs.github.io/helm-charts/docs/app-template/)
- [Diceware Passphrases](https://theworld.com/~reinhold/diceware.html)
