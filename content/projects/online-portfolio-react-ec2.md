---
title: "Deploying a Personal Portfolio: React on EC2 with Nginx"
date: 2026-02-15
description: "Taking a React portfolio site from a local build to a live, SPA-routed deployment on a bare Ubuntu EC2 instance — the manual deploy every later automation project builds on."
badges:
  - "AWS"
  - "EC2"
  - "React"
  - "Nginx"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-03-linux-and-bash-for-devops"
tags:
  - "AWS"
  - "EC2"
  - "React"
  - "Nginx"
toc: true
showInHome: false
image: "/images/online-portfolio-react-ec2/react-app.png"
deployStatus: "deployed"
statusLine: "React build → EC2 + Nginx · try_files SPA routing · zero orchestration"
---

Before any of the Terraform or Ansible automation elsewhere in this portfolio, I deployed my own portfolio site the manual way — a React app, a bare Ubuntu EC2 instance, and every step done by hand over SSH. It's the deployment every later automated version in this portfolio is really just re-encoding.

![Portfolio site rendering live from the EC2-hosted build](/images/online-portfolio-react-ec2/react-app-2.png)

## From a bare instance to a served build

Starting from a fresh Ubuntu EC2 instance with only SSH access, I installed Node.js and npm, then Nginx, then cloned the portfolio's React source. `npm install` pulled dependencies and `npm run build` compiled the app into a `/build` folder of plain, static HTML/CSS/JS — the only thing Nginx actually needs to serve. Deploying the raw source instead doesn't work: Nginx has no idea what to do with JSX or an unbundled `node_modules` tree, so the build step isn't optional, it's the entire point of the handoff between "React project" and "servable static site."

## SPA routing is the part that actually breaks

A single-page app has one entry point (`index.html`) and handles its own client-side routing in JavaScript — which means a plain Nginx config that 404s on any path other than `/` breaks the moment someone refreshes the page on a sub-route, or shares a direct link to it. The fix is a `try_files $uri /index.html;` directive in the Nginx server block: for any request that doesn't match a real file on disk, fall back to `index.html` and let React's router take over from there instead of letting Nginx return a 404 for a route it was never going to find as a file.

## Why this deployment matters beyond the lab

This exact pattern — build locally or on the instance, serve the compiled output through Nginx, get SPA routing right — is the manual baseline for two more automated versions elsewhere in this portfolio: the same React-on-Azure deployment done through Terraform (see [Azure Cloud Infrastructure]({{< relref "azure-cloud-infrastructure.md" >}})), and the fully declarative version provisioned end-to-end by `terraform apply` (see [Infrastructure as Code with Terraform]({{< relref "infrastructure-as-code-terraform.md" >}})). Doing it by hand first is what made the automated versions legible later — I knew exactly which manual steps each Terraform resource and Ansible task was standing in for.

## What I learned

- **The build step is the real deployment artifact, not the source code.** Every React deployment in this portfolio, manual or automated, exists to get a `/build` folder in front of Nginx correctly.
- **`try_files $uri /index.html` is the single line that makes a single-page app's client-side routing actually work in production** — without it, refreshing on any route but `/` is a 404 waiting to happen.
- **Doing a deployment by hand once is what makes automating it later actually mean something** — I could tell exactly which manual step a given Terraform resource or Ansible task was replacing, instead of trusting a template blindly.
