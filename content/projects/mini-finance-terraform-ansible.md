---
title: "Mini Finance: Provisioning to Deploy with Terraform and Ansible"
date: 2026-04-03
description: "A single Azure VM taken from terraform apply to a live static site, using a 4-play Ansible deployment that clones straight from GitHub."
badges:
  - "Terraform"
  - "Ansible"
  - "Azure"
  - "Nginx"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-09-ansible"
tags:
  - "Terraform"
  - "Ansible"
  - "Azure"
  - "Nginx"
toc: true
showInHome: false
image: "/images/mini-finance-terraform-ansible/mini-finance.png"
deployStatus: "deployed"
statusLine: "Terraform (6 resources) + Ansible (4 plays) · single Azure VM · SSH-key only"
---

Most of the Terraform-and-Ansible projects in this portfolio provision a fleet or a full application stack. Mini Finance is the deliberately small version of that same handoff — one VM, one static site — which made it the clearest place to see exactly where Terraform's job ends and Ansible's job begins.

![Mini Finance site live in the browser, served by Nginx on the Terraform-provisioned VM](/images/mini-finance-terraform-ansible/mini-finance-azure-architecture.svg)

## Terraform: the infrastructure half

A `mini-finance-rg` resource group in East US holds a `10.0.0.0/16` VNet with a single `10.0.1.0/24` subnet, a Network Security Group opening only SSH (22) and HTTP (80), a static Public IP bound to a Network Interface, and an Ubuntu 22.04 `Standard_B1s` VM (`mini-finance-vm`) provisioned with `admin_ssh_key` and `disable_password_authentication = true` — no password login exists on this box at all, only the key pair generated locally. `terraform apply` brings up all six resources and outputs the VM's public IP, which is the one manual handoff into the Ansible half.

## Ansible: four plays, each with one job

Rather than one long playbook, the deployment is split into four plays with narrow, single-purpose scopes:

1. **Install Nginx and Git** on the `[web]` host
2. **Clone and deploy the site** — `git` module pulls the static site from a forked GitHub repo into `/tmp`, then `copy` (with `remote_src: yes`) moves it into `/var/www/html`, owned by `www-data`
3. **Write the Nginx server block** — overwrites `/etc/nginx/sites-available/default` with a block pointing `root` at `/var/www/html` and `index` at `index.html`
4. **Restart Nginx** so the new server block and content take effect together, not mid-deploy

Splitting "install the server" from "deploy the content" from "restart" means a content-only update never needs to re-run the install step, and the restart is always the last, deliberate action — not an implicit side effect of an earlier task.

## The loop: apply, then playbook, then verify

The actual workflow is two commands with a manual handoff between them: `terraform apply` inside `terraform/`, copy the printed public IP into `ansible/inventory.ini`, then `ansible-playbook -i inventory.ini webserver-installation.yml` from `ansible/`. Verification is a plain `curl http://<public-ip>` (or a browser) — no orchestration layer, just confirming the chain actually worked end to end: DNS-free public IP → NSG → Nginx → `/var/www/html`.

## What I learned

- **A one-VM deployment is the clearest place to see the Terraform/Ansible boundary.** Terraform's job stops at "the VM exists and I can SSH into it"; everything after that — packages, content, service state — is Ansible's, and keeping that line clean scales cleanly to the larger fleet deployments elsewhere in this portfolio.
- **Splitting install, deploy, and restart into separate plays pays off even on a single host.** It's the same separation-of-concerns habit that matters more once you're running against a fleet instead of one box.
- **`disable_password_authentication` at the Terraform layer is a stronger guarantee than an Ansible-side SSH hardening task** — the password login path never exists in the first place, instead of being closed after the fact.
