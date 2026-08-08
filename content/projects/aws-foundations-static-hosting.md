---
title: "AWS Foundations: From Account Setup to Static Hosting on S3 & EC2"
date: 2026-02-15
description: "Building the Linux and AWS fundamentals — EC2 deployments, S3 static hosting, and a full production maintenance drill on a live Nginx server."
badges:
  - "AWS"
  - "EC2"
  - "S3"
  - "Nginx"
  - "Linux"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-03-linux-and-bash-for-devops"
tags:
  - "AWS"
  - "EC2"
  - "S3"
  - "Nginx"
  - "Linux"
toc: true
showInHome: false
image: "/images/aws-foundation-static-hosting/aws1.png"
deployStatus: "deployed"
statusLine: "EC2 + Nginx + S3 static hosting · ports 22/80 only · incident drill passed"
---

Before touching Terraform or Kubernetes, I spent time on the fundamentals that everything else sits on top of: a real Linux server, a real deployment, and the operational habits that keep it running. This case study covers three deployments that build on each other — a React app on EC2, a static site on S3, and a full production maintenance drill.

![EC2 deployment verified live in the browser](/images/aws-foundation-static-hosting/aws5.png)

## Deploying a React app on EC2 with Nginx

Starting from a bare Ubuntu EC2 instance, I installed Node.js, npm, and Nginx, cloned a React app, built the production bundle with `npm run build`, and served it through Nginx configured with SPA-friendly routing (`try_files $uri /index.html`). The full loop — SSH in, install runtime, build, deploy static assets to `/var/www/html`, configure the reverse proxy, verify in the browser — is the deployment pattern I reused across almost every later project.

## Static hosting on S3

Separately, I deployed a static portfolio site directly to an S3 bucket configured for website hosting: uploading the site contents (not the folder itself, so `index.html` stays at the bucket root), enabling static website hosting with `index.html`/`error.html`, and writing a bucket policy granting public `s3:GetObject` access. The distinction between an S3 *object* URL and the S3 *website endpoint* — and the fact that permissions, not just file placement, gate public access — was the main lesson here.

## Production maintenance drill

The most valuable part of this phase wasn't the deploy — it was a structured "ops checklist" I ran against the live EC2/Nginx server afterward, treating it like a real production system instead of a finished lab:

- **Networking & access** — `ip a`, `ip route`, DNS resolution, `ping`, and `ss -tulpen` to confirm only the expected ports (22, 80) were listening
- **Service health** — `systemctl status`, `systemctl is-enabled`, `nginx -t` config validation, and confirming the Nginx master/worker process model
- **Logs & request tracing** — generating real traffic with `curl`, then reading `access.log`, `error.log`, and the systemd journal to correlate what actually happened
- **Resource health** — `uptime`, `free -h`, `df -h`, and disk usage under `/var`, since log growth is one of the most common silent killers of a small VM
- **Incident simulation** — deliberately breaking the Nginx config (removing a semicolon) and the web root (moving it aside), confirming the failure mode (`nginx -t` fails, `curl` returns 404), then restoring service and documenting root cause → fix → prevention for each

## What I learned

- **A deployment isn't done when the browser loads it.** It's done when you've verified the service survives a reboot, a config error, and a missing file — and know how to recover from each.
- **`nginx -t` before every reload is non-negotiable.** It's the difference between a caught typo and unplanned downtime.
- **SSH keys over passwords, minimal open ports, and routine log review** are the baseline security habits that turned out to matter in every later project, not just this one.
