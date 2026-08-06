---
title: "Azure Cloud Infrastructure: VM Provisioning to Load-Balanced 3-Tier Networks"
date: 2026-03-05
description: "From a single Azure VM serving a React app to a load-balanced, zone-redundant 3-tier network with a public and internal load balancer."
badges:
  - "Azure"
  - "Load Balancer"
  - "Networking"
  - "Nginx"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-07-azure-cloud"
tags:
  - "Azure"
  - "Networking"
  - "Load Balancing"
  - "High Availability"
toc: true
showInHome: false
---

This case study tracks how I scaled Azure infrastructure knowledge from a single manually-provisioned VM up to a proper multi-tier, load-balanced network — the Azure counterpart to the AWS work in this portfolio, and the foundation the later Terraform automation builds on.

## Single VM: React on Azure

The starting point was deliberately manual: a Resource Group, a B1s Ubuntu VM with SSH-key auth and NSG rules restricting port 22 to my own IP while opening port 80, then Node.js, Nginx, and a React production build served with SPA-aware routing. I hardened it afterward with **Fail2Ban** to automatically block IPs after repeated failed SSH attempts — a layer beyond just restricting the port.

## Designing a 3-tier network with a load balancer

The next step moved beyond a single box to a segmented network: a VNet (`10.0.0.0/16`) split into web, application, and database subnets, with **Azure Bastion** enabled for VM access that never exposes SSH directly to the internet. A public Load Balancer was configured with a frontend IP, a backend pool containing the web-tier VM, and a health probe on port 80 — so traffic reaches the application only through the load balancer, never a direct VM IP.

## EpicBook on a VM with Azure Database for MySQL

I then deployed a real full-stack app — EpicBook, a Node.js bookstore — onto this pattern: a VNet with public/private subnets, NSG rules for SSH/HTTP/app traffic, Node.js + Nginx on the VM, and **Azure Database for MySQL** (a managed service, not a self-hosted database) holding the `bookstore` schema. Nginx reverse-proxied port 80 to the app on port 8080. The first deploy hit a 502 Bad Gateway — the backend simply wasn't running yet — which is a useful reminder that a reverse proxy returning 502 almost always means "upstream not listening," not a proxy misconfiguration.

## Capstone: zone-redundant 3-tier architecture

The most advanced version of this pattern was a full production-shaped deployment of a Book Review app across two Availability Zones:

- **Six subnets** across 2 AZs (web / app / db × 2), each tier with its own NSG enforcing least-privilege inbound rules
- **Two load balancers**: a public LB fronting the web tier (Next.js behind Nginx, PM2-managed), and an **internal** LB — no public IP — routing web-tier requests to the app tier (Node/Express, also PM2-managed) on port 3001
- **App tier VMs have no public IP at all**; they're reachable only through the internal load balancer, and JWT secrets are pulled from **Azure Key Vault via Managed Identity** rather than stored in plaintext
- **Azure Database for MySQL Flexible Server**, zone-redundant with automatic failover, a read replica in the secondary zone, SSL-enforced connections, and 7-day point-in-time-restore backups — reachable only from the app tier's NSG, over a private FQDN, never a public endpoint

Schema import order mattered here: seeding data before the schema was fully applied threw foreign-key constraint errors, resolved by strictly sequencing schema-first, then seed data in dependency order.

## What I learned

- **A load balancer isn't just for scale — it's what makes "no public IP on the backend" possible.** The internal LB pattern is the actual mechanism behind tier isolation, not just a diagram convention.
- **Managed identity + Key Vault beats environment-variable secrets** the moment you're running more than one instance of anything.
- **Zone redundancy is a configuration, not an afterthought** — it has to be designed into the subnet layout and the database tier from the start, not bolted on later.
