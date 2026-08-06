---
title: "Production CI/CD Pipelines with Azure DevOps"
date: 2026-04-14
description: "A self-hosted Azure DevOps agent and three real deployment pipelines — static site, React build, and a two-repo infra/app split for EpicBook."
badges:
  - "Azure DevOps"
  - "CI/CD"
  - "Terraform"
  - "Ansible"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-10-azure-devops"
tags:
  - "CI/CD"
  - "Azure DevOps"
  - "Pipelines"
toc: true
showInHome: false
---

Manual deployment gets you a working app; a pipeline gets you a *repeatable* one. This project moved through three pipelines of increasing complexity, all running on a self-hosted agent I set up myself rather than relying on Microsoft-hosted build minutes.

## Self-hosted agent, from scratch

Before any pipeline could run, I needed compute to run it on. I generated a Personal Access Token, created an agent pool, provisioned an Azure VM, installed the Azure Pipelines agent on it, registered it against the pool using the PAT, and validated it with a test pipeline running basic Linux commands (`uname -a`, `whoami`, `df -h`). This mattered beyond the lab exercise — a later pipeline hit `No hosted parallelism has been purchased or granted`, a real limitation on new Azure DevOps organizations, and having a self-hosted agent already running as a systemd service was the actual fix, not a support ticket.

## Pipeline 1: static site to a Linux VM

The first real pipeline deployed a static website automatically on every push to `main`. Terraform provisioned the infrastructure (resource group, VNet, public IP, NSG, NIC, Ubuntu VM) — including discovering that Azure rejects underscores in a VM's `computer_name`, so the Azure resource name and the OS hostname needed separate fields. Ansible then configured the VM (Nginx, `/var/www/html` ownership), hitting an `sshpass`-missing error that clarified *why* Ansible needs that utility: to supply SSH passwords non-interactively, since automation can't respond to a manual prompt. The pipeline itself was a straightforward four-step YAML: checkout → list files → copy over SSH → verify remotely, secured by an **SSH Service Connection** in Azure DevOps so credentials never live in the YAML.

## Pipeline 2: a proper multi-stage React pipeline

The second pipeline was structured as four dependent stages — **Build → Test → Publish → Deploy** — each running on a fresh agent, passing outputs forward as artifacts. The key insight: Nginx cannot serve React source files. `npm run build` compiles JSX and bundles assets into a `/build` folder of plain HTML/CSS/JS, and *that* is what gets deployed — deploying raw source simply doesn't work. Infrastructure came from Terraform (VNet, subnet, public IP, NSG, NIC), configuration from an Ansible playbook fixing file ownership on `/var/www/html` so the pipeline's SSH deploy step could write without permission errors. Two sharp edges here: the SSH service connection's name in the Azure DevOps portal has to exactly match the `sshEndpoint` value in the pipeline YAML, and reprovisioning a VM with Terraform changes its SSH host key — triggering `REMOTE HOST IDENTIFICATION HAS CHANGED`, fixed cleanly with `ssh-keygen -R` rather than blanket-disabling host key checking.

## Pipeline 3: two-repository infra/app split for EpicBook

The most enterprise-shaped version of this pattern split infrastructure from application concerns into **two separate repositories and two separate pipelines**: `infra-epicbook` holds Terraform for the VNet, subnets, public IPs, and frontend/backend VMs, authenticating via an **App Registration Service Principal** and publishing the app's public IP and MySQL FQDN as pipeline outputs. `theepicbook` holds the Ansible playbooks, retrieving the SSH private key from **Azure DevOps Secure Files** and dynamically rewriting the Ansible inventory using the Terraform outputs *before* running the playbook — the actual handoff mechanism between infrastructure-as-code and configuration management in a pipeline context, rather than something stitched together by hand.

## What I learned

- **Self-hosted agents aren't just a workaround — they're what unlimited pipeline minutes and full environment control actually looks like** in a real organization, not just a free-tier limitation.
- **Stages run on fresh agents; artifacts are the correct way to pass build output between them** — you can't assume state persists across stage boundaries.
- **The infra/app repository split is what makes Terraform outputs consumable by Ansible in an automated pipeline** — Secure Files and pipeline variable groups are the plumbing that makes that handoff safe.
