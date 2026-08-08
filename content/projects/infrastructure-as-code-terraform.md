---
title: "Infrastructure as Code with Terraform"
date: 2026-03-20
description: "Multi-cloud VM provisioning with Terraform — from a single Azure VM to a 13-resource three-tier EpicBook stack on AWS, with every failure documented."
badges:
  - "Terraform"
  - "AWS"
  - "Azure"
  - "IaC"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-08-terraform"
tags:
  - "Terraform"
  - "Infrastructure as Code"
  - "AWS"
  - "Azure"
toc: true
showInHome: false
image: "/images/infrastructure-as-code-terraform/epicbook.png"
deployStatus: "provisioned"
statusLine: "terraform apply · 6 → 13 resources · AWS + Azure, zero manual steps"
---

Terraform is where the manual, portal-driven provisioning from the AWS and Azure case studies in this portfolio turns into repeatable, version-controlled infrastructure. This project moved through four increasingly complex builds, each one adding a layer: single VM → networked VM with a web server → full application deployment → multi-tier production architecture.

## Progression

**Azure VM (6 resources).** A single `main.tf` defining resource group, VNet, subnet, public IP, NIC, and an Ubuntu 18.04 VM — `terraform init` → `plan` → `apply` → verify via Azure CLI → `terraform output` for the public IP → `terraform destroy`. The full lifecycle in one file.

**AWS VM with public networking (8 resources).** A custom VPC, public/private subnets, an Internet Gateway, route table, a security group, and an EC2 instance that installs Nginx automatically via `user_data` on first boot — no manual SSH step required before the server is live.

**React app on Azure via Terraform (10 resources).** Provisioning plus application deployment in one pass: after `terraform apply`, I SSH'd in, installed Node 18, built the React app, and configured Nginx — with the Terraform-managed static public IP ensuring the address didn't change between reboots.

**EpicBook on AWS (13 resources).** A VPC with a public subnet for EC2 and *two* private subnets for RDS (AWS requires Multi-AZ subnet groups even for a single-AZ instance), security groups referencing each other rather than open CIDR ranges, and a full app deploy — clone, `npm install`, configure the DB connection from the Terraform-output RDS endpoint, PM2, Nginx reverse proxy. I verified it end-to-end through the actual application flow: browsing, adding to cart, checkout — confirming the database write path worked, not just that the server responded.

## Real errors, real fixes

Terraform's error messages were the actual curriculum here:

- **`SkuNotAvailable`** — the requested VM size was out of capacity in the target region; fixed by switching region and/or size rather than retrying blindly.
- **Region-mismatched AMI ID** — an AMI copied from a us-east-1 example doesn't exist in eu-west-2; AMIs are region-specific even when the OS is identical.
- **"Provider produced inconsistent result"** on subnet/public IP resources — a known AzureRM provider bug, resolved by pinning the provider version (`3.85.0`) and setting `skip_provider_registration = true`.
- **A resource group stuck deleting for 6+ minutes** during `terraform destroy` — resolved via `az group delete --yes --no-wait` directly through the CLI, then clearing local state before reapplying.
- **RDS security group misconfiguration** — initially opened by CIDR block; corrected to reference the EC2 security group ID directly (`security_groups = [aws_security_group.ec2_sg.id]`), so only that specific instance can reach the database, regardless of its IP.

## Going further: a modular capstone

![Book Review app three-tier architecture, provisioned entirely by Terraform modules](/images/infrastructure-as-code-terraform/book-review-app-aws-architecture.svg)

The most advanced build used a proper module structure (`modules/networking`, `modules/security`, `modules/ec2`, `modules/database`, `modules/alb`) to provision a two-AZ, six-subnet, dual-load-balancer three-tier architecture for a Book Review app — the same shape as the manually-built HA architecture elsewhere in this portfolio, but fully declarative and reproducible from `terraform apply`.

## What I learned

- **`terraform destroy` is as important as `apply`.** Every build in this project was fully torn down and verified as zero-resource-remaining, because idle cloud resources are the easiest way to burn a free-tier budget.
- **Terraform errors are almost always either capacity, region-mismatch, or provider-version issues** — once you recognize the pattern, debugging speeds up dramatically.
- **`count` + a list variable eliminates repetition** — provisioning N near-identical resources from one block is both less code and less room for copy-paste drift.
