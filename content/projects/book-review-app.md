---
title: "Book Review App: Three-Tier Architecture Across AWS, Azure, and Terraform"
date: 2026-03-12
description: "One three-tier Book Review app — Next.js, Node/Express, MySQL — deployed as the capstone across the AWS, Azure, and Terraform case studies in this portfolio."
badges:
  - "AWS"
  - "Azure"
  - "Terraform"
  - "Next.js"
  - "Load Balancing"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-06-aws-cloud"
tags:
  - "Book Review App"
  - "AWS"
  - "Azure"
  - "Terraform"
  - "High Availability"
toc: true
showInHome: false
image: "/images/book-review-app/book-review-app-aws-architecture.svg"
deployStatus: "deployed"
statusLine: "same 3-tier architecture · 3 ways: AWS, Azure, Terraform modules"
---

The Book Review app is this portfolio's other recurring application — where EpicBook proves the same app deployed *differently* across clouds, Book Review proves the same three-tier, load-balanced *architecture* built three different ways: manually on AWS, manually on Azure, and declaratively through Terraform modules.

![Book Review app running on the three-tier capstone deployment](/images/book-review-app/book-review-app.png)

## The architecture, held constant across every version

Every version of this deployment keeps the same shape: a **public load balancer** in front of a web tier (Next.js behind Nginx, PM2-managed) that never talks to the database directly, and an **internal load balancer** — no public IP — routing web-tier requests to a backend tier (Node/Express, also PM2-managed). The database sits behind the app tier only, reachable from nowhere else. What changes between versions is *how* that shape gets built and *which* cloud it runs on.

## Three builds of the same shape

**Manually on AWS** — the capstone of the [Highly Available & Three-Tier Architecture]({{< relref "highly-available-aws-architecture.md" >}}) case study: 4 subnets across 2 Availability Zones, a public ALB and an internal ALB, RDS MySQL Multi-AZ with a read replica, and an Auto Scaling Group backing the web tier. Debugging here was mostly reverse-proxy and environment routing — a books-not-displaying bug traced to `.env.local`, a backend 500 traced to Nginx not forwarding `Host`/`X-Real-IP` headers.

**Manually on Azure** — the capstone of the [Azure Cloud Infrastructure]({{< relref "azure-cloud-infrastructure.md" >}}) case study: the same public-LB/internal-LB split, but with app-tier VMs holding no public IP at all, JWT secrets pulled from **Azure Key Vault via Managed Identity** instead of plaintext, and **Azure Database for MySQL Flexible Server** running zone-redundant with SSL-enforced, private-FQDN-only connections.

**Declaratively via Terraform** — the modular capstone of [Infrastructure as Code with Terraform]({{< relref "infrastructure-as-code-terraform.md" >}}): the identical two-AZ, six-subnet, dual-load-balancer shape, but provisioned from `modules/networking`, `modules/security`, `modules/ec2`, `modules/database`, and `modules/alb` — reproducible from a single `terraform apply` instead of built by hand through the portal.

## What holding the architecture constant taught me

Building the same three-tier shape three separate ways made the *portable* parts of the design obvious versus the *cloud-specific* parts: the public/internal load-balancer split, tier isolation, and least-privilege security-group chaining carried over unchanged every time; what changed was entirely provider-specific plumbing — ALB vs. Azure Load Balancer target groups, RDS Multi-AZ vs. Azure Database for MySQL Flexible Server, IAM roles vs. Managed Identity.

## What I learned

- **An internal load balancer is what actually makes "no public IP on the backend" possible**, on AWS or Azure — it's a mechanism, not a diagram convention, and it's the one piece of this architecture that has to exist regardless of cloud.
- **Rebuilding the same architecture on a second cloud is a better test of whether you understood it than building two different architectures once each.** The parts that transferred cleanly were the parts I'd actually understood; the parts that needed provider-specific research were the parts I'd only pattern-matched.
- **A Terraform module structure is the same architectural decisions, just made explicit and reusable** — `modules/alb` and `modules/database` aren't new design choices, they're the manual AWS and Azure builds written down properly.
