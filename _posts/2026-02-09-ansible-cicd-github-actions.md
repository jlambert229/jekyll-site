---
title: "Ansible CI/CD with GitHub Actions"
subtitle: "Run ansible-lint and check mode on every PR. Catch config drift and risky changes before they hit production."
date: 2026-02-09
tags:
  - ansible
  - cicd
  - github-actions
  - linting
categories:
  - Homelab Infrastructure
readtime: true
---
Every PR that touches Ansible playbooks should pass `ansible-lint` and `ansible-playbook --check`. GitHub Actions runs them automatically - no "I'll lint before merge" discipline required.
<!--more-->

> *"If it's not in CI, it doesn't exist."*

*"That's a big no."* - *Alien*. CI is the chestburster that stops bad config before it reaches production.

---

## Why CI for Ansible?

- **ansible-lint** - Catches deprecated patterns, undefined variables, risky shell usage, YAML issues
- **Check mode** - Dry run: what *would* change? Catches logic errors, missing vars, unreachable hosts
- **PR gate** - Block merge until lint and check pass. Prevents "works on my machine" configs

I pushed a playbook change once. "ansible-lint passed locally." It didn't. I'd run it on a different branch. CI caught a `command` instead of `shell` with pipe characters - a security concern I'd missed. The PR sat blocked until I fixed it. Embarrassing. Also exactly what CI is for. Let the robots catch the dumb mistakes.

> *"The goal is to make the right thing the easy thing."* - Don Norman, and every good CI design

---

## Workflow

```yaml
# .github/workflows/ansible-ci.yml
name: Ansible CI

on:
  pull_request:
    paths:
      - '**.yml'
      - '**.yaml'
      - 'roles/**'
      - 'inventory/**'
      - '.github/workflows/ansible-ci.yml'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install Ansible and ansible-lint
        run: |
          pip install ansible ansible-core
          pip install ansible-lint

      - name: Install collections
        run: ansible-galaxy collection install -r requirements.yml

      - name: Run ansible-lint
        run: ansible-lint playbooks/ roles/ inventory/

  check:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install Ansible
        run: pip install ansible ansible-core

      - name: Install collections
        run: ansible-galaxy collection install -r requirements.yml

      - name: Check mode (dry run)
        run: ansible-playbook playbooks/site.yml --check
        env:
          ANSIBLE_HOST_KEY_CHECKING: 'false'
        # Note: Check mode needs reachable hosts. Use localhost or mock inventory
        # if your hosts aren't reachable from GitHub.
```

---

## The host problem

Check mode connects to real hosts. In CI, your Proxmox nodes usually aren't reachable from GitHub's runners.

**Options:**

### 1. Use a localhost playbook for check mode

Create a minimal play that only validates syntax and variable resolution:

```yaml
# playbooks/check-syntax.yml
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Validate variable resolution
      debug:
        msg: "{{ backup_schedule }}"
      vars:
        backup_schedule: "{{ backup_schedule | default('0 2 * * *') }}"
```

Then run `ansible-playbook playbooks/check-syntax.yml` against a static inventory with `localhost`. No real hosts needed.

### 2. Limit to lint only

If hosts aren't reachable, skip check mode in CI. Lint still catches most issues:

```yaml
- name: Run ansible-lint
  run: ansible-lint playbooks/ roles/ inventory/
```

### 3. Self-hosted runner with network access

Run the workflow on a runner in your homelab. It can reach Proxmox hosts. Heavier setup, but full check mode against real inventory.

---

## Path filters

Run only when Ansible files change:

```yaml
on:
  pull_request:
    paths:
      - '**.yml'
      - '**.yaml'
      - 'roles/**'
      - 'inventory/**'
```

Saves CI minutes. Ignores docs-only PRs.

---

## ansible-lint config

Create `.ansible-lint` to tune rules:

```yaml
# .ansible-lint
profile: production
exclude_paths:
  - .github/
strict: true
```

---

## What I learned

> *"The sooner you find a bug, the cheaper it is to fix. CI finds bugs before merge."* - Cost of delay, quantified

**Lint catches real bugs.** Unbelievable but true: I had a task with `command: "{{ some_var }}"` and `some_var` could be undefined in certain edge cases. Lint flagged it. I thought "that's overly cautious." Check mode would have failed at runtime. On a host. During a patch run. At 2am. Lint is not overly cautious. Lint is your friend. Listen to lint.

**Check mode needs inventory.** Even with `--check`, Ansible resolves inventory and tries to connect. My first CI run: "Host unreachable." The GitHub runner couldn't SSH to my Proxmox hosts. Of course it couldn't. Use `--limit localhost` or a CI-only inventory. Or accept lint-only. Your runners live in the cloud. Your hosts don't.

**Path filters are your friend.** I fixed a typo in the README. CI ran. Four minutes. For a typo. Proxmox-ops changes rarely. Filter to `**.yml`, `roles/**`, `inventory/**`. README edits don't need the full pipeline. Save the minutes. Save your patience.

---

## References

- [ansible-lint documentation](https://ansible.readthedocs.io/projects/lint/)
- [Ansible check mode](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html#cmdoption-ansible-playbook-C)
- [Managing Proxmox with Ansible](/post/ansible-proxmox-ops/)
