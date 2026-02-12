---
title: "Ansible Vault for Secrets"
subtitle: "Encrypt API keys, backup passwords, and tokens in group_vars with ansible-vault. Keep secrets in Git without exposing them."
date: 2026-02-09
tags:
  - ansible
  - security
  - vault
  - secrets
categories:
  - Homelab Infrastructure
readtime: true
---
API keys, backup passwords, Proxmox tokens. You need them in Ansible. You don't want them in plaintext in Git. Ansible Vault encrypts files so you can commit them safely.
<!--more-->

> *"Secrets in version control? Only if they're encrypted."*

*"Don't panic."* - Hitchhiker's Guide. And don't commit plaintext. Encrypt it with ansible-vault.

---

## Why Vault?

Plaintext `secrets.yml` gets committed. Someone pushes to a public fork. Credential leak. Or you're on a shared machine and `group_vars/` gets backed up to an unencrypted NAS.

**Vault encrypts files with AES-256.** Ansible decrypts them at runtime with your password or key file. The repo holds ciphertext, not plaintext.

| Scenario | Without Vault | With Vault |
|----------|---------------|------------|
| `git push` | Secrets in history | Encrypted blob |
| Stale backup | Plaintext recoverable | Useless without key |
| PR review | Reviewer sees secrets | Reviewer sees `$ANSIBLE_VAULT` |

> *"Secrets management is defense in depth. One layer fails; the next catches it."* - Security engineering 101

---

## Setup

### 1. Create the secrets file

```yaml
# inventory/group_vars/proxmox_hosts/secrets.yml
# Values here override or supplement proxmox_hosts.yml

# Backup notifications (optional)
backup_mailto: "admin@example.com"

# Proxmox API token (for Terraform, PBS, etc.)
proxmox_api_token: "root@pam!terraform=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# NFS credentials if using auth (most homelabs use IP allowlist)
# nfs_options: "vers=3,soft,intr,user=backup,password={{ backup_nfs_password }}"
# backup_nfs_password: "your-nfs-password"
```

### 2. Encrypt it

```bash
ansible-vault encrypt inventory/group_vars/proxmox_hosts/secrets.yml
# Enter a strong password when prompted
```

Ansible replaces the content with an encrypted block. The file is safe to commit.

### 3. Ensure it's loaded

Ansible loads all `*.yml` in a group_vars directory. If `secrets.yml` lives in `group_vars/proxmox_hosts/`, it's loaded with the rest. No extra config.

### 4. Run with decryption

```bash
# Prompt for vault password each run
ansible-playbook playbooks/site.yml --ask-vault-pass

# Or use a password file (keep it out of Git!)
echo 'your-vault-password' > ~/.ansible-vault-pass
chmod 600 ~/.ansible-vault-pass
ansible-playbook playbooks/site.yml --vault-password-file ~/.ansible-vault-pass
```

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Vault password file</div>
<div class="admonition-content" markdown="1">

Add `~/.ansible-vault-pass` to `.gitignore` if it's in the repo directory. Never commit the password file.

</div>
</div>

---

## Directory layout

For proxmox-ops:

```
inventory/group_vars/proxmox_hosts/
├── proxmox_hosts.yml    # Non-sensitive config (plaintext)
└── secrets.yml          # Encrypted with ansible-vault
```

Split by sensitivity: plain config in one file, secrets in another. Encrypt only what's sensitive.

> *"The best secret is the one that never leaves the vault."* - And the second-best is encrypted before it hits Git.

---

## Common operations

```bash
# Edit encrypted file (decrypt, edit, re-encrypt)
ansible-vault edit inventory/group_vars/proxmox_hosts/secrets.yml

# View without editing
ansible-vault view inventory/group_vars/proxmox_hosts/secrets.yml

# Re-key (change password)
ansible-vault rekey inventory/group_vars/proxmox_hosts/secrets.yml

# Decrypt (e.g., to migrate to a different secrets store)
ansible-vault decrypt inventory/group_vars/proxmox_hosts/secrets.yml
```

---

## CI/CD with Vault

GitHub Actions (or similar) needs the vault password to run playbooks. Store it as a repository secret:

1. Repo → Settings → Secrets → `ANSIBLE_VAULT_PASSWORD`
2. In the workflow:

```yaml
- name: Run playbook
  env:
    ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
  run: |
    echo "$ANSIBLE_VAULT_PASSWORD" > .vault-pass
    chmod 600 .vault-pass
    ansible-playbook playbooks/site.yml --vault-password-file .vault-pass
    rm .vault-pass
```

---

## What to put in secrets

| Variable | Why |
|----------|-----|
| `proxmox_api_token` | Terraform, API-based automation |
| `backup_mailto` | SMTP credentials if backup uses mail |
| `pve_admin_email` | Let's Encrypt ACME if enabled |
| NFS/API passwords | Any credential passed to roles |

Keep structure (keys, variable names) in plain `proxmox_hosts.yml`. Put values in `secrets.yml`.

---

## What I learned

> *"The best secret is the one that never leaves the vault."* - And the second-best is encrypted before it hits Git.

**Start with one file.** I encrypted the entire `group_vars` with ansible-vault once. Editing was a nightmare. Every `ansible-vault edit` decrypted everything. A typo in the non-secret config meant re-encrypting the whole blob. Now I only encrypt the file with credentials. Easier to edit. Easier to audit. Easier to not throw things.

**Document the template.** I came back to this repo after six months. "What do I put in secrets.yml?" No example. No instructions. I had to dig through playbooks to reverse-engineer the variable names. Add `secrets.yml.example`. Your future self - or the person who inherits this - will thank you.

**Password file for local dev.** Typing the vault password 47 times during a debugging session is a special kind of torment. `--vault-password-file` saves your sanity. Use `--ask-vault-pass` only for one-offs. And for penance.

**The panic moment.** I once encrypted `secrets.yml` with ansible-vault and pushed to Git. Forgot to add `~/.ansible-vault-pass` to `.gitignore`. It lived in the repo directory. I realized mid-push. Pulled the cable. Too late - it was already on the remote. Spent the next hour rotating every credential in that file, re-keying the vault, and updating `.gitignore` while my heart rate slowly returned to normal. The file itself wasn't committed (I'd caught it), but the *risk* of it was enough. Vault password file in `.gitignore`. Always. Check twice.

---

## References

- [Managing secrets with Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- [Managing Proxmox with Ansible](/2026-02-09-ansible-proxmox-ops/) - Parent article
- [CI/CD with GitHub Actions](/2026-02-09-ansible-cicd-github-actions/) - Passing vault password in CI
- [Scheduled Ansible runs](/2026-02-09-ansible-scheduled-runs/) - Cron needs `--vault-password-file`