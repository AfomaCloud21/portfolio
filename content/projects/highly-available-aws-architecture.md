---
title: "Highly Available & Three-Tier Architecture on AWS"
date: 2026-03-12
description: "A VPC/ALB/ASG/Multi-AZ RDS deployment with proven failover, capped by a three-tier Book Review app spanning public and internal load balancers."
badges:
  - "AWS"
  - "VPC"
  - "ALB"
  - "Auto Scaling"
  - "RDS"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-06-aws-cloud"
tags:
  - "AWS"
  - "High Availability"
  - "RDS"
  - "Load Balancing"
toc: true
showInHome: false
image: "/images/highly-available-aws-architecture/epicbook.png"
deployStatus: "deployed"
statusLine: "VPC + ALB + ASG (2–4 instances) · Multi-AZ RDS failover · 2 failure tests passed"
---

Deploying an app is one thing; proving it survives failure is another. This project builds a highly-available two-tier AWS architecture and then verifies it under simulated failure conditions, before scaling the pattern up to a full three-tier capstone.

## EpicBook on EC2 + private RDS

The first iteration deployed EpicBook (Node.js + MySQL) into a custom VPC (`10.0.0.0/16`) with a **public subnet** for the EC2 app server and a **private subnet** for RDS — the database has no route to the internet at all. Security groups enforced the chain: SSH (22) and HTTP (80) reach EC2 from allowed sources; MySQL (3306) reaches RDS *only* by referencing the EC2 security group, not by opening a CIDR range. Node.js ran under PM2 for persistence across SSH sessions, with Nginx reverse-proxying to it — the same 403/502 debugging pattern (wrong root path, missing reverse proxy headers) showed up here as it did on the Azure equivalent of this deployment.

## Building in redundancy: VPC, ALB, ASG, Multi-AZ RDS

The next iteration made the architecture genuinely fault-tolerant rather than just network-isolated:

- **4 subnets across 2 Availability Zones** — 2 public (ALB + web tier), 2 private (RDS) — with an Internet Gateway for public traffic and a NAT Gateway so private-subnet instances can still reach package repositories
- **Chained security groups**: ALB SG allows HTTP from anywhere → EC2 SG allows HTTP only from the ALB SG (plus SSH from my IP) → RDS SG allows the database port only from the EC2 SG
- **RDS Multi-AZ** for automatic failover to a standby in a second AZ
- **A Launch Template with user data** so new EC2 instances self-configure — installing the web server and wiring the DB connection automatically, without manual SSH setup
- **An Auto Scaling Group** (min 2 / desired 2 / max 4) attached to the ALB's target group, using ELB health checks so unhealthy instances get replaced automatically

### Proving it, not just diagramming it

I ran two failure tests rather than trusting the architecture on paper: terminating one web instance and confirming the ASG launched a replacement while the ALB kept serving traffic without interruption, then simulating an AZ disruption by stopping an instance in one zone and confirming the other zone absorbed the load with no downtime.

## Capstone: three-tier Book Review app

![Three-tier Book Review app architecture on AWS](/images/highly-available-aws-architecture/book-review-app-aws-architecture.svg)

The final iteration split the architecture into three properly isolated tiers behind two load balancers — a public ALB in front of the web tier (Next.js + Nginx), and an **internal** ALB routing only from the web tier to backend EC2 instances (Node.js + PM2) in private subnets, backed by RDS MySQL Multi-AZ with a read replica. Real production debugging showed up here: a books-not-displaying bug traced to incorrect environment routing (fixed by rebuilding the frontend after correcting `.env.local`), a backend 500 traced to Nginx not forwarding `Host`/`X-Real-IP` headers, and a private-subnet package install failure traced to missing outbound routing — resolved via correct security-group referencing and bastion-style access.

## What I learned

- **High availability is a claim you have to test, not a diagram you draw.** Killing an instance and watching the ASG/ALB recover is the only real proof.
- **Security group chaining (SG-to-SG references) is safer and more maintainable than CIDR-based rules** — it scales with the architecture instead of needing manual IP updates.
- **Most "AWS bugs" in a multi-tier app are actually reverse-proxy header or environment-variable bugs** — the infrastructure was usually right; the request path through it wasn't.
