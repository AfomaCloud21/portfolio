---
title: "Agile DevOps Lifecycle with Jira & Scrum"
date: 2026-02-10
description: "Running a full Scrum lifecycle solo — backlog, sprints, and a 5-day shipped increment — for a portfolio site deployed to EC2."
badges:
  - "Jira"
  - "Scrum"
  - "Agile"
  - "EC2"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/2134_gotto_job-5-"
tags:
  - "Jira"
  - "Scrum"
  - "Agile"
  - "DevOps Lifecycle"
toc: true
showInHome: false
---

DevOps isn't just pipelines and infrastructure — it's also the delivery process wrapped around them. This project was about proving I could run that process end to end: standing up a Scrum board in Jira, breaking work into epics and stories, running a real sprint, and shipping a verifiable increment to production on a deadline.

## What I built

I set up a Team-managed Scrum project in Jira for a small web property ("Gotto Job"), then ran the full cycle solo — deliberately taking on every Scrum role at once:

- **Product Owner** — prioritized UI-only improvements that would build user trust
- **Scrum Master** — followed ceremonies, tracked blockers, kept the board honest
- **Dev Lead** — implemented small, visible changes without touching the backend
- **DevOps Lead** — owned deployment to EC2 and shipped proof for every change

The backlog work included an Epic ("Improve Gotto Job UI discoverability & trust"), 8 user stories with Fibonacci estimates and testable acceptance criteria, and a Sprint 1 scoped to 4 stories / 6 story points — deliberately small enough to actually finish.

## The 5-day sprint

For an earlier iteration of this exercise (a DevOps Micro-Internship portfolio site), I ran a tighter 5-day mini-sprint and shipped one increment per day, each one committed, deployed to EC2, and verified live:

- **Day 1** — implemented a footer (version, deploy date, author) and deployed
- **Day 2** — made the deploy date dynamic instead of hardcoded, updated the README with evidence
- **Day 3** — polish and accessibility pass, verified on desktop and mobile via DevTools
- **Day 4** — replaced the tagline with a call-to-action link
- **Day 5** — recorded a demo, ran the retro, reviewed the burndown chart, closed the sprint

Every day's work followed the same loop: commit with a meaningful message → push → deploy to EC2 → verify on the live URL. Small, traceable, production-validated increments rather than one large end-of-sprint drop.

## Shipping under real constraints

The "Gotto Job" sprint didn't go smoothly, and that was the more instructive run. Deployment on Day 8 failed repeatedly — SCP errors traced back to the project folder being nested inside an unexpected parent directory, compounded by an EC2 instance that had drifted into an unstable state from missing configuration. Rather than patch around it, I deleted the instance, provisioned a fresh one, re-pulled the project, reorganized the folder structure properly, and redeployed. It cost the better part of a day, but it's the kind of failure that teaches you to *rebuild* rather than *fight* broken infrastructure.

## What I learned

- **Sprint goals are a filter, not a target.** Committing to less and delivering all of it beats overcommitting and missing.
- **Solo Scrum still creates accountability.** Explicitly separating Product Owner / Scrum Master / Dev / DevOps thinking — even as one person — forced better decisions than just "doing tasks."
- **Deployment failures are usually structural, not mysterious.** File layout and instance drift caused more downtime here than any application bug did.
- **Retrospectives are where the actual improvement happens.** Writing down what broke and why is what made the next sprint faster.
