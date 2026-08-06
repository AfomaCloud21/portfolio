---
title: "DMI Cohort 2 Capstone: Petcare Clinic CloudOps"
date: 2026-06-12
description: "Co-leading CI/CD and Jira delivery for an 11-person team shipping Spring PetClinic on AWS EKS with Terraform, Docker, GitOps via ArgoCD, and full observability — then redeploying the same stack solo."
badges:
  - "Kubernetes"
  - "AWS EKS"
  - "Terraform"
  - "ArgoCD"
  - "GitHub Actions"
  - "Microservices"
links:
  - icon: fab fa-github
    url: "https://github.com/Petcare-Clinic/petcareClinic-cloudops"
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/spring-petclinic-microservices"
tags:
  - "Kubernetes"
  - "GitOps"
  - "CI/CD"
  - "Microservices"
  - "Observability"
  - "Terraform"
toc: true
---

This was the final capstone of the DevOps Micro Internship: an 11-person, globally distributed team (Nigeria and India) took the open-source [Spring PetClinic Microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) application and deployed it as a real, production-shaped system on AWS — Terraform-provisioned EKS, Dockerized services, GitHub Actions CI, ArgoCD GitOps CD, and a full Prometheus/Grafana/Zipkin observability stack — in a **7-day execution window**. I co-led the project alongside our Cloud lead, owning Jira delivery and the CI/CD pipeline end to end. I followed it with a solo redeployment of the same application, to prove I understood the whole system, not just my slice of it — a redeployment that ended up teaching me as much about adapting under a real resource constraint as the original team build did.

## Architecture

Requests enter through an **API Gateway** (Spring Cloud Gateway) that handles routing, rate limiting, and circuit breaking. Behind it, a **Config Server** serves centralized, environment-specific configuration to every other service, and a **Discovery Server** (Eureka) keeps a live registry of every microservice instance — when `vets-service` needs `customers-service`, it asks Eureka for the current address rather than relying on a hardcoded IP. Three domain services — **Customers**, **Vets**, **Visits** — each own an isolated MySQL database, so a schema failure in one never cascades into another. **Zipkin** provides distributed tracing across the whole request path, **Prometheus** scrapes metrics from every service's `/actuator` endpoint every 15 seconds, and **Grafana** turns those metrics into dashboards.

```
Users → NGINX Ingress (TLS via cert-manager) → API Gateway
                                                     │
                        ┌────────────┬───────────────┼───────────────┬────────────┐
                   Config Server  Discovery      Customers         Vets         Visits
                                  (Eureka)        + MySQL         + MySQL      + MySQL
                                                     │
                        Prometheus · Grafana · Zipkin (observability layer)
```

## Team structure and my role

The team split into six functional groups — Project Leads, Cloud Engineering, Docker/Containerization, Kubernetes, CI/CD, and Monitoring — each owning a distinct slice of the stack. As **Project Lead & CI/CD Engineer**, my responsibilities spanned two very different disciplines:

**Delivery ownership** — built the Jira Scrum board from scratch (6 epics, 34 user stories, 2 sprints with story points and assignees across all 10 other engineers), onboarded the whole team to Jira/GitHub/Discord on day one, ran daily standups and both sprint retrospectives, and defined the `main` / `dev` / `feature-*` branch strategy with protection rules requiring PR review before any merge to `main`.

**CI/CD ownership** — designed and implemented the GitHub Actions + ArgoCD pipeline that took every commit from `git push` to a running pod on EKS with zero manual steps, managed the GitHub Secrets it depended on (AWS credentials, ECR registry, EKS cluster name, ArgoCD server), and installed and configured ArgoCD itself on the cluster.

## The 7-day build

| Day | Focus |
|---|---|
| 1 | Jira board, epics, team onboarding, GitHub org setup, branch strategy |
| 2 | System architecture design, service dependency mapping, Kubernetes deployment strategy |
| 3 | Terraform init, EKS planning, ECR repository creation, IAM configuration |
| 4 | Containerizing all services, local Docker builds and testing |
| 5 | GitHub Actions CI pipeline — build automation, image automation, push-to-ECR automation |
| 6 | ArgoCD installation, GitOps deployment wiring, namespace configuration |
| 7 | End-to-end testing, monitoring validation, deployment verification, demo prep |

## Jira, on an interface that had changed under us

Most Jira tutorials online were written against the old UI; the current Jira Cloud interface had moved things significantly — the Epic panel is hidden by default in Team-managed projects, the backlog structure had changed, and even locating sprint configuration took real exploration rather than following a guide. I got through it by reading the current documentation directly, testing configuration changes manually, and collaborating with teammates hitting the same walls — which taught me more about actually navigating unfamiliar tooling under time pressure than following a tutorial ever would.

## Standing up AWS from a WSL2 control machine

Working from WSL2 meant the AWS CLI wasn't preinstalled — I installed it manually (`curl` the installer, `unzip`, `sudo ./aws/install`), configured credentials with `aws configure`, and verified authentication with `aws sts get-caller-identity` before touching any infrastructure. Running `aws eks list-clusters` early on returned an empty list — useful negative information, confirming AWS access and region config were correct while making clear the EKS cluster genuinely didn't exist yet and infrastructure work had to land before anything downstream could deploy. Once the cluster was up, `aws eks update-kubeconfig` wired `kubectl` to it, and ECR repositories were created per service (`aws ecr create-repository --repository-name spring-petclinic-config-server`) ahead of the first image push.

## The bug that mattered most: Docker image naming

The CD pipeline failed at the tagging step with a flat `No such image` error, even though Maven had just built every service successfully. The pipeline expected an image named `petclinic-config-server`; Maven's build plugin had actually produced `springcommunity/spring-petclinic-config-server` — a completely different name, not just a missing tag. Rather than guess again, I ran `docker images | grep petclinic` to print every image Maven had actually built, matched the tag step to the real name, and the push to ECR succeeded immediately after. That one command — print the actual state instead of assuming it — is now a debugging habit I reach for before touching any pipeline config.

## When the cluster itself was the problem

After the CD pipeline started succeeding, the application still wasn't reachable — pods stuck `Pending`, ArgoCD stuck `OutOfSync`, services unable to communicate. Working through `kubectl get nodes`, `kubectl get pods -A`, and `kubectl get svc -A` traced it to the EKS worker nodes themselves not running correctly, which meant nothing could schedule regardless of how correct the manifests were. I used Claude AI to help work through the diagnosis systematically — checking node readiness, verifying namespaces, reinstalling ArgoCD's components, and restarting the affected services — until nodes came back healthy and ArgoCD could sync cleanly. **Kubernetes deployment failures are so often node/cluster health, not manifest content** — check the foundation before debugging the YAML.

## Tearing it down cleanly

Infrastructure ownership doesn't stop at `apply` — `terraform destroy` needed to work too, and at first it didn't. A teardown attempt stalled with the VPC refusing to delete: residual ECR images were still referenced inside it, and a stale security group (left behind from an earlier failed run) had a dependency Terraform couldn't resolve on its own. I cleared the ECR repositories manually, removed the orphaned security group, and only then re-ran `destroy` to completion. It's a small thing that matters at real cost: leftover cloud resources from a botched teardown are exactly how a demo project turns into a surprise bill.

## Git at team scale: merge conflicts and divergent branches

With 11 people editing overlapping files, merge conflicts were routine rather than exceptional — `CONFLICT (content): Merge conflict in README.md` type errors, resolved by reviewing the conflicting sections manually, comparing branch differences, and keeping the correct blocks (Claude AI helped analyze the more tangled ones). A separate class of error — `fatal: Need to specify how to reconcile divergent branches` — came from pulling after someone else had already pushed; `git pull --rebase origin Dev` resolved it cleanly by replaying local commits on top of the remote history instead of creating a merge commit.

## CI/CD: from push to a running pod, in ~8 minutes

The CI workflow (`.github/workflows/ci.yml`) runs on every push: checkout, JDK/Maven setup, `./mvnw clean install`, unit tests. The CD workflow (`cd.yml`) is a three-job pipeline: build and push Docker images to ECR tagged by branch (`dev-<run#>` off `Dev`, `v1.0.<run#>` off `main`), update the corresponding Helm values file and commit with `[skip ci]` (paired with a `paths-ignore` rule on the values files, so the CD workflow's own commits can't re-trigger itself), then trigger an ArgoCD sync and wait up to 300 seconds for health. ArgoCD watches the repo independently and reconciles Kubernetes to match — the actual GitOps loop: **Git is the source of truth; the cluster continuously converges to whatever it says.** Across dev, staging, and production, the team ended up running 6 active pipelines (main CI/CD, PR validation, dev deploy, staging deploy, production deploy, and a manual/automatic rollback that reverts a bad release via `helm rollback` in under 30 seconds), with production gated behind required GitHub Environment reviewers.

## The solo redeployment: when Docker Compose wasn't enough

Weeks later, for the individual half of this capstone, I redeployed the same application myself — Spring Petclinic, all core services plus the observability stack — starting with Docker Compose (`docker compose up -d`, one command, every container). The first blocker was environmental: Docker Desktop's WSL2 integration wasn't enabled for my distro, so Compose failed before it ever reached the application, fixed via Docker Desktop → Settings → Resources → WSL Integration.

The second blocker didn't have a settings-panel fix. Running the full stack — Config Server, Eureka, three domain services, API Gateway, plus Prometheus, Grafana, and Zipkin — simultaneously on an 8GB-RAM laptop exceeded available memory. Rather than keep fighting Compose on hardware it wasn't going to fit on, I adapted the deployment strategy in real time and deployed the same application independently to an **EC2 instance** instead, where I had headroom to run every service without starving the host machine. Compose is the right tool when your hardware fits the stack; when it doesn't, moving the whole thing to a right-sized cloud instance is a faster fix than trying to trim services out of a demo that's supposed to prove the full system works.

The lesson underneath both attempts was the same one from the ArgoCD/EKS incident, just at a different layer: **startup order is not incidental.** Config Server holds every other service's configuration; Eureka is the registry every service needs to find its neighbors. If anything else starts before those two are actually *healthy* — not just running — it fails to fetch config or register and comes up broken. `depends_on` with health checks is what enforces "wait until ready," not "wait until the container exists," on EC2 just as much as it was on EKS.

I verified the final deployment across three layers: the browser (owners, pets, visit history, and vets list all functional), an automated `curl` HTTP-200 check, and the observability stack itself confirming registration and traceability — watching a request flow through `api-gateway → customers-service` live in Zipkin made distributed tracing concrete in a way no architecture diagram had.

## What I'd change for real production

Replace the Compose file with EKS manifests for local-to-cloud parity; put the Application Load Balancer and Ingress in front of every environment, not just production; move secrets into a real secrets manager instead of `.env` files; add persistent storage for Prometheus and Zipkin (both run in-memory by default, losing history on every pod restart); and extend branch protection and required reviewers down to `staging`, not just `main`.

## What I learned

- **Delivery is a team sport.** The architecture, infrastructure, and monitoring stack weren't individually mine — but getting all of it to converge, deploy on time, and hold up under a mentor demo belonged to all eleven of us.
- **A cluster health check comes before a manifest review.** The EKS node incident cost real time because the instinct was to debug YAML first; `kubectl get nodes` should have been step one.
- **AI-assisted debugging is a skill, not a shortcut.** Every time Claude helped resolve an issue — the ArgoCD incident, the merge conflicts — understanding *why* the fix worked well enough to explain it back was what made it actually useful rather than a black box.
- **The same failure mode shows up at every layer if you're not careful about it.** The CD pipeline's image-naming bug, the EKS node incident, and the Compose stack's startup-order dependency were all the same lesson — verify actual state, don't assume it — surfacing in three completely different parts of the same system.
- **Infrastructure ownership includes teardown, and knowing when to abandon a plan.** A blocked `terraform destroy` is still my responsibility to resolve, not just the `apply`. And when local hardware genuinely can't run the stack, the right move is switching environments, not shrinking the demo to fit — that's the difference between a workaround and actually solving the problem.
