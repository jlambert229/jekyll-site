---
title: "Vault + Consul: Enterprise Secret and Config Management for Kubernetes"
subtitle: "Migrating from hardcoded secrets to Vault and Consul with automated backups. Secrets in Vault, config in Consul, zero hardcoded values."
date: 2026-02-09
tags:
  - kubernetes
  - vault
  - consul
  - security
  - secrets
  - configuration
  - external-secrets
  - backup
categories:
  - Kubernetes Homelab
readtime: true
---
I found my Gmail password in Git. In a public repo. Three months after I pushed it.
<!--more-->

Not a commit from years ago that I'd forgotten about. A commit from *last Tuesday*. In `apps/monitoring/values.yaml`:

```yaml
alertmanager:
  config:
    global:
      smtp_auth_password: 'your-app-password'  # right there in plaintext
```

The commit was public. The repo was public. The password was public. And I had no idea until I was browsing GitHub on my phone and saw it.

> *"Secrets in Git are not a matter of if they leak - they're a matter of when."* - Every security audit ever

*"The password is... oh."* - Every developer who found their Gmail token in a public repo. Time to fix this properly.

## The Options

I looked at three approaches:

**Sealed Secrets** - Encrypt secrets, commit them to Git. Still didn't like having encrypted blobs in my repo. What if there's a crypto vulnerability someday? What if I misconfigure something?

**SOPS** - Similar idea, different tool. Better key management, but same fundamental issue: encrypted secrets in Git.

**Vault** - Secrets live in Vault. External Secrets Operator syncs them to K8s. Git has zero secrets.

I went with Vault. Also added Consul for configuration management (PUID, timezone, ports - stuff that isn't secret but also shouldn't require Git commits to change).

> *"Secrets and config are different problems. Secrets: who can access. Config: what values to use. Don't conflate them."* - Secrets management principle

Is it overkill for a homelab? Absolutely. But I was trying to learn production patterns, and this is what production does.

---

## How It Works

**Vault** stores secrets (passwords, API keys, tokens). External Secrets Operator authenticates to Vault using Kubernetes service accounts, pulls secrets, creates K8s Secrets. Apps consume those Secrets like normal. They don't know Vault exists.

**Consul** stores configuration (ports, resource limits, timezone, etc.). Init containers fetch config on startup and write it to files. Apps read those files. Again, apps don't know Consul exists.

**Backups** run daily. Export Vault secrets and Consul config to NFS. Keep 30 days of history.

The apps themselves don't change. They still read K8s Secrets and config files. We just changed where those come from.

---

## Deploying Vault

I used dev mode. Don't do this in production - it runs in-memory with auto-unsealing and a hardcoded root token - but for homelab it's perfect. No persistent storage setup, no unsealing process, just works.

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault -n vault --create-namespace --set server.dev.enabled=true
```

Root token is `root`. Data is in-memory. Pod restart = data loss. That's fine - I set up automated backups later.

### Setting Up Vault

Vault needs some config: enable KV secrets engine, set up Kubernetes authentication, create policies. I automated this with a Job that runs after Vault starts:

```bash
vault login root
vault secrets enable -path=secret kv-v2
vault auth enable kubernetes
# ... policy and role creation
```

### External Secrets Operator

This is the bridge between Vault and Kubernetes. It authenticates to Vault, pulls secrets, creates K8s Secrets automatically.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

Configure a SecretStore to tell it how to reach Vault:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: media
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          role: "external-secrets"
```

### Moving Secrets to Vault

I exec'd into the Vault pod and started populating secrets:

```bash
kubectl exec -n vault vault-0 -it -- sh
vault login root
vault kv put secret/monitoring/smtp username='my-email@gmail.com' password='...'
vault kv put secret/monitoring/pushbullet token='...'
vault kv put secret/monitoring/grafana password='...'
```

Three secrets moved from Git to Vault. Repository is now clean.

---

## Syncing to Kubernetes

Create ExternalSecret resources. ESO watches Vault and creates K8s Secrets automatically:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: smtp-credentials
  namespace: media
spec:
  refreshInterval: "1h"
  secretStoreRef: {name: vault-backend}
  target: {name: smtp-credentials}
  data:
  - secretKey: smtp_password
    remoteRef: {key: monitoring/smtp, property: password}
```

```bash
kubectl apply -f externalsecrets.yaml
kubectl get externalsecrets -n media  # Should show SecretSynced
```

ESO creates a K8s Secret called `smtp-credentials`. Apps consume it like any other Secret. They don't know it came from Vault.

## Updating Apps

The apps didn't change. Before, they read secrets from hardcoded Helm values. Now they read the same secrets from K8s Secrets (which ESO populates from Vault).

Changed `apps/monitoring/values.yaml` from:

```yaml
smtp_auth_password: 'your-app-password'  # ❌ hardcoded
```

To:

```yaml
# SMTP password in Vault: secret/monitoring/smtp
env:
  SMTP_PASSWORD:
    secretKeyRef: {name: smtp-credentials, key: smtp_password}
```

Redeployed. Worked. No app changes needed.

## Rotating Secrets

Now when I need to rotate a password:

```bash
kubectl exec -n vault vault-0 -- vault kv put secret/monitoring/smtp password='NEW_PASSWORD'
```

ESO syncs it within an hour. Or I delete the K8s Secret to force immediate sync. No Git commit. No redeployment. Just update Vault, restart pods.

## Consul for Configuration

Secrets were solved. But I had other problems: configuration values hardcoded everywhere. PUID, timezone, ports, resource limits - all in Git.

Deployed Consul, populated it with 37 config values:

```bash
consul kv put config/media/common/puid "1000"
consul kv put config/media/common/timezone "America/New_York"
consul kv put config/media/apps/sonarr/port "8989"
# ... 34 more
```

Apps fetch config via init containers on startup. Change timezone for all apps:

```bash
consul kv put config/media/common/timezone "Europe/London"
kubectl rollout restart -n media deployment/sonarr  # picks up new value
```

No Git commit needed. Same pattern as Vault: centralize, fetch at runtime, never hardcode.

## Automated Backups

Vault runs in dev mode (in-memory). Pod restart = data loss. So backups are critical.

CronJob runs daily at 2AM:

```bash
vault kv export -format=json secret/ > backup.json
gzip backup.json
# Store on NFS, keep 30 days
```

Same for Consul:

```bash
consul kv export config/ > backup.json
consul snapshot save snapshot.snap
# Compress, store, 30-day retention
```

I've tested restore. It works. That's all that matters.

## What I Learned

**Dev mode is fine for homelab.** Zero time spent on storage, unsealing, HA. It just works. Yes, data loss on pod restart. Backups run daily. I tested restore. The alternative is unsealing rituals and persistent volumes and "why isn't Vault starting?" at 11pm. Dev mode. Accept it.

**ESO is invisible to apps.** They consume K8s Secrets. They don't know Vault exists. Don't care. No code changes. I migrated three apps. Changed nothing except removing hardcoded secrets. ESO created the Secrets. Apps kept working. It's almost too easy. Almost.

**Rotation is trivial.** Update a value in Vault. Wait an hour. Or force sync. Done. No Git commits. No deployments. No "did I push that?" I rotated my SMTP password. Changed it in Vault. ESO propagated. Alertmanager picked it up. I did nothing else. Black magic. Good black magic.

**Test your backups.** I assumed they worked. They didn't. The backup script had a bug - it was backing up an empty directory. I discovered this when I needed to restore. I no longer assume. I restore quarterly. Paranoid? Maybe. Prepared? Definitely.

**Document the paths.** Three months later I won't remember if it's `secret/monitoring/smtp` or `secret/smtp/monitoring`. I had to grep my own scripts to find a path. Comment everything. Your memory is not as good as you think.

## The Result

Before: 3 secrets hardcoded in Git. Public repo. Anyone could see them.

After: Zero secrets in Git. All in Vault. Synced to K8s automatically. Rotatable without commits. Backed up daily.

Configuration moved from Git to Consul. Change timezone across 9 apps with one command, no deployment.

Repository is public now. No secrets, no embarrassment. Everything's properly managed.

## Worth It?

For homelab? Probably overkill. For learning production patterns? Absolutely.

The setup took a weekend. Vault + ESO + Consul + backups. Now:
- Rotate secrets without Git
- Change config without deployments
- Sleep better knowing nothing sensitive is public
- Actually learn how production does this

Would I do it again? Yes. Would I recommend it for everyone? No. If you're just running Plex for yourself, hardcoded values in a private repo are fine.

But if you're learning, if you care about security, or if your repo might ever be public - do this properly from the start.

---

## Related

- [Ansible Vault](/2026-02-09-ansible-vault-secrets/) - Encrypt secrets in Ansible; different use case, same principle
- [Velero Backups](/2026-02-08-k8s-velero-backups/) - Backup Vault data and Consul config

Full implementation: [k8s-media-stack](https://github.com/YOUR-USERNAME/k8s-media-stack) (see `foundation/vault/`, `foundation/consul/`, `secrets/`, `backup/`)
