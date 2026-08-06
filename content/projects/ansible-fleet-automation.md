---
title: "Configuration Management & Fleet Automation with Ansible"
date: 2026-04-05
description: "Provisioning 4 Azure VMs with Terraform, wiring passwordless SSH at scale, and moving from ad-hoc commands to multi-play, idempotent deployments."
badges:
  - "Ansible"
  - "Terraform"
  - "Azure"
  - "SSH"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-09-ansible"
tags:
  - "Ansible"
  - "Configuration Management"
  - "Automation"
  - "Terraform"
toc: true
showInHome: false
---

Terraform builds the infrastructure; Ansible configures it. This project covers that handoff at increasing scale — from four bare VMs to a production-style, role-based deployment — and the free-tier quota walls I had to design around along the way.

## Provisioning a 4-VM fleet under real quota constraints

Terraform provisioned four Ubuntu VMs across **two Azure regions**, not because the architecture called for it, but because free-tier subscriptions cap public IPs at 3 per region and vCPUs at 4 per region — a `Standard_D2s_v3` fleet of 4 VMs simply doesn't fit in one region. Splitting `web1`/`web2` into Canada Central and `app1`/`db1` into West US, with independent networking stacks per region, was the fix. Azure also only accepts RSA SSH keys, not the `ed25519` key the assignment originally specified, so the key generation step became `ssh-keygen -t rsa -b 4096` with the public half injected via Terraform's `admin_ssh_key` block — giving passwordless SSH to all four VMs the moment `apply` finished.

## From ad-hoc commands to a real inventory

With `inventory.ini` grouping the fleet into `[web]`, `[app]`, and `[db]`, I worked through Ansible's ad-hoc mode as a diagnostic and fleet-management tool: `ansible all -m ping` to verify SSH + Python on every host in one line, group-targeted commands (`ansible web -m apt ... --become` to install Nginx only on web hosts, `ansible db -m command -a "df -h"` to check disk only on the database host), and confirming **idempotency** directly — re-running the same `apt` install returns `OK`, not `CHANGED`, once the package is already present.

## Multi-play playbooks and separation of concerns

Moving from one-liners to a real playbook, I split a static-site deployment into three plays with distinct responsibilities: install/configure Nginx, deploy content via the `copy` module, then verify from the control node using `uri`. Splitting install from deploy wasn't just tidiness — it means Nginx installation runs once, while content deployment can run dozens of times a day without re-touching the server setup, each with its own handler scope so a content change doesn't trigger an unrelated service restart.

## Production-shaped deployment: role-based EpicBook

The most advanced build separated concerns properly using **Ansible roles** — `common` (baseline packages, SSH hardening), `nginx` (reverse proxy config via Jinja2 template), and `epicbook` (clone, Node.js, MySQL, PM2, database seeding) — deployed against Terraform-provisioned infrastructure with the inventory auto-updated by a `null_resource` `local-exec` block. I proved idempotency the same way production teams do: running the playbook a second time and confirming `changed=0 failed=0`.

Real debugging, not lab debugging: a Git "dubious ownership" error running as root against a `www-data`-owned directory (fixed with `git config --global --add safe.directory`); a 403 that turned out to be Nginx trying to serve a dynamic app as static files (fixed with `proxy_pass` instead of `root`); MySQL `Access denied` errors from Ubuntu 22.04's `auth_socket` plugin not playing well with Ansible's `become` shell (fixed by using the Ubuntu-provided `debian-sys-maint` account instead of fighting root socket auth); and PM2 showing no processes after SSH because `PM2_HOME` set during automation didn't persist into an interactive shell.

## What I learned

- **Cloud free-tier quotas are architectural constraints, not edge cases** — designing around a 3-IPs-per-region cap taught me to read quota errors as design input, not just obstacles to retry past.
- **Ansible's idempotency is the whole point.** A playbook you can't safely re-run isn't a playbook, it's a script that happens to work once.
- **Ad-hoc commands and playbooks solve different problems** — one-off checks and emergency fixes vs. version-controlled, repeatable, auditable deployments — and knowing which one a situation calls for is itself the skill.
