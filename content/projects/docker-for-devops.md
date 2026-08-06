---
title: "Docker for DevOps"
date: 2026-04-25
description: "Multi-stage builds that cut image size by 89-95%, and a full production-grade EpicBook container stack with healthchecks, backups, and CI/CD."
badges:
  - "Docker"
  - "Docker Compose"
  - "CI/CD"
  - "Nginx"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-11-docker"
tags:
  - "Docker"
  - "Containers"
  - "CI/CD"
toc: true
showInHome: false
---

Containerizing an app correctly is less about "does it run in a container" and more about what you leave out. This project moved from a basic cloud-init Docker bootstrap to multi-stage builds with measured size reductions, and finally to a full production-shaped stack for EpicBook with healthchecks, backups, and a real CI/CD pipeline.

## Bootstrapping Docker with cloud-init

Rather than SSH in and install Docker manually after provisioning, I embedded a cloud-init script directly in the Terraform configuration for the VM, so Docker is installed and enabled automatically on first boot. Deployment became: provision → SSH in with key auth → clone → build the Nginx-based container image → run mapped to port 80.

## Multi-stage builds: the numbers

The clearest lesson in this project was quantified, not theoretical. A single-stage React build image retains Node.js, npm, the full `node_modules` tree, and the entire Webpack/Babel toolchain — none of which the running app needs:

| Metric | Single-stage | Multi-stage | Reduction |
|---|---|---|---|
| Uncompressed size | 2.15 GB | 94 MB | 95.4% |
| Compressed size (registry push/pull) | 497 MB | 26.3 MB | 94.7% |

The multi-stage version uses a `node:18-alpine` builder stage to compile the app, then `COPY --from=builder` pulls only the finished static bundle into a fresh `nginx:alpine` runtime stage — no interpreter, no package manager, no build toolchain left for an attacker to abuse even with shell access. It also compounds operationally: a 26 MB image pushes and pulls in seconds where a 497 MB one takes minutes, on every commit, every branch, every replica.

The other optimization that mattered: copying `package.json`/`package-lock.json` and running `npm ci` *before* copying source files. Docker caches each instruction as its own layer — if the manifest hasn't changed, that layer is reused untouched, so a typical code-only change rebuilds in ~10 seconds instead of ~3 minutes, because the ~300MB dependency install step never re-runs. One easy-to-miss gotcha: `CMD ["nginx"]` lets Nginx fork into the background as it does by default, so Docker sees the parent process exit and kills the container immediately; `CMD ["nginx", "-g", "daemon off;"]` keeps it in the foreground so Docker can actually track and signal it.

## EpicBook: the production-grade container stack

The capstone build took EpicBook from "runs in a container" to a system I'd trust operationally:

- **Multi-stage Dockerfile** (`node:20-alpine`) — 123MB final image vs. ~1.1GB single-stage baseline, an 89% reduction, running as a non-root user
- **Three isolated services, two Docker networks** — `frontend_net` connects Nginx to the app; `backend_net` connects the app to MySQL. Nginx has no network path to the database under any configuration.
- **Healthchecks with real startup ordering** — a `/health` endpoint backed by `db.sequelize.authenticate()`, with `depends_on: condition: service_healthy` guaranteeing MySQL → app → Nginx comes up in that order every time, not just "usually"
- **Verified persistence** — 54 books survive a full `docker compose down` / `up` cycle, backed by an automated daily `mysqldump` backup with gzip compression and 7-backup rotation
- **Structured logging** — JSON access logs on a named volume, Docker's `json-file` driver with 10MB rotation, queryable with `jq`
- **CI/CD via a self-hosted Azure Pipelines agent** — build stage pushes tagged images to Azure Container Registry; deploy stage pulls the new image, restarts only the app container, and waits for healthy status before declaring success. 28 pipeline runs completed, triggered automatically on every push to `main`.
- **Reliability tests, not just deploys** — restarting the app alone produced a brief 502 that self-recovered; stopping the database flipped `/health` to 503 and recovered automatically once the DB returned; a full stack bounce preserved all 54 books.

## What I learned

- **A container image's size is a security surface, not just a disk-space number** — every package left in is something a scanner (or an attacker) can find a CVE against.
- **Healthchecks plus `depends_on: condition: service_healthy` is what actually enforces startup order** — without it, "the database container is running" and "the database is ready for connections" get conflated, and the app fails intermittently on cold starts.
- **A deploy that doesn't wait for the health check before declaring success isn't a real deploy** — it's a race condition wearing a green checkmark.
