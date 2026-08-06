---
title: "EpicBook: End-to-End Multi-Cloud Production Deployment"
date: 2026-04-20
description: "One Node.js bookstore app, deployed and redeployed across AWS, Azure, Docker, Terraform, Ansible, and CI/CD — the flagship thread running through this whole portfolio."
badges:
  - "AWS"
  - "Azure"
  - "Terraform"
  - "Ansible"
  - "Docker"
  - "CI/CD"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/app-deploy-epicbook"
tags:
  - "EpicBook"
  - "Terraform"
  - "Ansible"
  - "Docker"
  - "Multi-Cloud"
toc: true
---

EpicBook — a Node.js + MySQL bookstore app — is the one project that shows up across nearly every DevOps discipline in this portfolio. Rather than write it up once, this case study pulls together its most complete iteration: **Terraform + Ansible on Azure**, cross-referenced against its AWS, Docker, and CI/CD variants elsewhere in this site.

## Architecture

A two-phase automated pipeline: Terraform provisions the Azure infrastructure, Ansible configures the VM and deploys the app. Requests flow through a standard reverse-proxy chain — Nginx (port 80, the only public entry point) → Node.js/Express on port 8080 (PM2-managed) → MySQL on port 3306 (local, never exposed). The NSG firewall opens only ports 22 and 80; MySQL is unreachable from outside the VM entirely.

## Infrastructure and configuration, cleanly separated

**Terraform (7 resources)** — resource group, VNet, subnet, static public IP, NSG, NIC, and the Ubuntu 22.04 VM — with a `null_resource` `local-exec` block that automatically updates `inventory.ini` and `~/.ssh/config` the moment `apply` finishes, so passwordless SSH (`ssh epicbook-vm`) works immediately without a manual step.

**Ansible, as three roles** — `common` (baseline packages, SSH hardening: disabled root login, disabled password auth), `nginx` (installs Nginx, deploys a Jinja2 reverse-proxy vhost, disables the default site), and `epicbook` (clones the repo, installs Node.js 18 and MySQL, bootstraps the database password, seeds the schema, starts the app under PM2).

I proved the deployment was production-safe the way it actually matters: re-running `ansible-playbook` a second time and confirming `changed=0 failed=0` — full idempotency, not just "it worked once."

## Five real production bugs, and the actual fixes

- **Git "dubious ownership"** running as root against a `www-data`-owned directory → `git config --global --add safe.directory` before the clone task.
- **403 Forbidden in the browser** → Nginx was trying to serve the app as static files (`root` + `index`); switched to `proxy_pass http://localhost:8080` since EpicBook is a dynamic Express app, not a static site.
- **MySQL `Access denied` on every bootstrap attempt** → Ubuntu 22.04's `auth_socket` plugin doesn't cooperate with Ansible's non-login `become` shell; fixed by using the `debian-sys-maint` account Ubuntu creates specifically for automated MySQL administration, sidestepping root socket auth entirely.
- **PM2 showing an empty process list after SSH** → Ansible starts PM2 with `PM2_HOME=/root/.pm2`, but an interactive `azureuser` SSH session defaults to a different path; fixed with a shell alias pinning `PM2_HOME` consistently.
- **App loaded with no books** → schema and seed data need explicit, ordered import (`community.mysql.mysql_db` with `state: import`), guarded by a table-count check so re-running the playbook doesn't re-seed.

## The same app, other ways

This exact application also shows up, deployed differently, across the rest of this portfolio: on **AWS EC2 with a private-subnet RDS instance** (see [Highly Available & Three-Tier Architecture on AWS]({{< relref "highly-available-aws-architecture.md" >}})); provisioned entirely through **Terraform on both AWS and Azure** (see [Infrastructure as Code with Terraform]({{< relref "infrastructure-as-code-terraform.md" >}})); fully **containerized with Docker Compose, healthchecks, and a CI/CD pipeline pushing to Azure Container Registry** — 3 containers, 89% smaller image via multi-stage builds, 28 completed pipeline runs (see [Docker for DevOps]({{< relref "docker-for-devops.md" >}})); and behind a **two-repository infra/app Azure DevOps pipeline** with Terraform outputs feeding Ansible inventory automatically (see [Production CI/CD Pipelines with Azure DevOps]({{< relref "production-cicd-azure-devops.md" >}})).

## Security gaps I documented rather than shipped around

This is a learning deployment, and I treated the gap between "works" and "production-ready" as something to name explicitly: SSH open to `0.0.0.0/0` instead of a restricted range, plain HTTP with no TLS, MySQL credentials in plaintext Ansible variables instead of Ansible Vault, `root`-privileged database access instead of a scoped app user, no monitoring or alerting, and a single VM with no redundancy. Each has a specific, named fix — Vault encryption, Let's Encrypt, a dedicated DB user, Azure Monitor, a load balancer — rather than a vague "would add security later."

## What I learned

- **The same application deployed five different ways is a better way to learn a tool than five different toy apps** — the constants (reverse proxy pattern, DB isolation, idempotent config) become obvious, and the tool-specific parts stand out clearly.
- **Ubuntu's `debian-sys-maint` account is the actual supported path for automated MySQL admin** — fighting root `auth_socket` from within Ansible is a dead end every time.
- **Naming the security gaps you didn't fix is itself a production skill** — it's the difference between "this works" and "this is what I'd change before this touches real users."
