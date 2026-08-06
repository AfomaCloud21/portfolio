---
title: "Kubernetes Fundamentals"
date: 2026-04-29
description: "From a single imperative Pod to a public LoadBalancer service on AKS — ReplicaSets, Deployments, HPA, readiness/liveness probes, and Services, each proven by deliberately breaking it."
badges:
  - "Kubernetes"
  - "AKS"
  - "Azure"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-12-kubernetes"
tags:
  - "Kubernetes"
  - "AKS"
  - "Containers"
toc: true
---

The fastest way to actually understand a Kubernetes concept is to break it on purpose and watch how the cluster reacts. Every stage of this project followed that pattern: build the feature, then deliberately misconfigure it to see the failure mode before fixing it.

## Pods, ReplicaSets, Deployments

Starting from the smallest unit, I created a Pod both **imperatively** (direct `kubectl` commands — fast, good for quick tasks) and **declaratively** (a YAML manifest defining desired state — how Kubernetes is actually meant to be run in production, since it continuously reconciles actual state against what's declared). A standalone Pod has an obvious gap: delete it, and it's gone. A **ReplicaSet** fixes that by maintaining a fixed replica count — deleting a managed Pod triggers immediate, automatic replacement, Kubernetes' self-healing in action. But ReplicaSets can't do rolling updates, which is exactly the gap **Deployments** fill: updating the image version triggered a gradual, zero-downtime replacement of old Pods with new ones, and I also exercised **rollback** to a previous version — the safety net that makes shipping updates low-risk.

## Auto-scaling and health probes

With a Deployment running multiple replicas, I configured a **Horizontal Pod Autoscaler** to add or remove Pods automatically based on CPU utilization within a defined min/max range — scaling out under load, back in when demand drops, without manual intervention.

**Readiness probes** answer *"can this Pod serve traffic right now?"* — until yes, Kubernetes marks it `NotReady` and excludes it from traffic. I intentionally broke a readiness probe on a Deployment with `maxUnavailable: 0`, and the rollout **stalled correctly**: new Pods weren't considered ready, so Kubernetes refused to terminate the still-healthy old ones — exactly the zero-downtime guarantee readiness probes exist to provide.

**Liveness probes** answer a different question: *"should this container be restarted?"* — catching the case where a process is technically running but has silently deadlocked or leaked memory into unresponsiveness. I pointed a liveness probe at an invalid endpoint on purpose and watched the restart count climb, then corrected it and watched the restart cycle stop. Readiness gates traffic; liveness gates survival — they solve different failure modes and both are necessary.

## Services: from internal discovery to a public endpoint

Healthy, scalable Pods still have a problem: Pod IPs are ephemeral, so nothing can reliably address them directly. A **ClusterIP Service** solves this with a stable virtual IP and DNS name, using label selectors to track which Pods currently qualify — I confirmed this by deliberately breaking the selector (traffic stopped) and fixing it (traffic resumed immediately), and observed that `NotReady` Pods are automatically excluded from Service endpoints, tying readiness probes directly into routing behavior. A **NodePort** Service layered external access on top — the same ClusterIP behavior, plus a fixed port opened on every node — useful without a cloud load balancer, but reliant on knowing specific node IPs and non-standard ports.

The final step moved to a real public endpoint: a **LoadBalancer** Service on **Azure Kubernetes Service (AKS)**. Changing a single field — `type: LoadBalancer` — was enough for Azure to provision a public IP and cloud load balancer automatically, with no manual infrastructure work. Breaking the readiness probe again here showed the health-awareness runs the *entire* chain, cloud load balancer down to individual Pod — unhealthy Pods dropped out of rotation even from the public internet. Scaling from 2 to 4 replicas during this test left the external IP completely unchanged; only the pool of eligible backends grew, exactly the behavior you want from production traffic.

The trickiest issue wasn't Kubernetes at all — it was environmental: `az aks get-credentials` wrote the kubeconfig to a Windows path that Git Bash couldn't read, so `kubectl` kept silently falling back to a local `kind` cluster instead of AKS. Fixed with the `--file` flag pointing at a Linux-accessible path and merging kubeconfigs via the `KUBECONFIG` environment variable.

## What I learned

- **Readiness and liveness probes are not the same control**, and conflating them is a common way to either serve traffic to a broken Pod or restart a Pod that was actually fine.
- **A Service's stability comes from labels and selectors, not from tracking Pod IPs** — get the selector wrong and the abstraction silently breaks.
- **`type: LoadBalancer` is a one-line declaration that triggers real cloud provisioning behind the scenes** — worth understanding what it actually does before relying on it in production.
