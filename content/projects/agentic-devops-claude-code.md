---
title: "Agentic DevOps with Claude Code"
date: 2026-04-10
description: "Standing up a full agentic AI workflow for DevOps work — CLAUDE.md, custom Skills, purpose-built Subagents, MCP servers, and hook-based safety controls."
badges:
  - "Claude Code"
  - "Agentic AI"
  - "MCP"
  - "Automation"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-02-agentic-ai"
tags:
  - "Claude Code"
  - "Agentic AI"
  - "DevOps Tooling"
toc: true
---

Traditional DevOps tooling is fragmented by design — Terraform for infrastructure, GitHub for code, Docker for containers, monitoring in yet another tool, with a human doing all the coordinating. This project was about building the opposite: a single agentic workflow where Claude Code plans, acts, and verifies across that whole toolchain, with explicit safety controls so "agentic" doesn't mean "unsupervised."

## Teaching Claude the project: CLAUDE.md

The first layer is context. Running `/init` against a project generates a baseline `CLAUDE.md`, but the generated version is generic — I rewrote it to be specific: the actual HTML/CSS + AWS S3/CloudFront architecture, a documented folder structure, explicit conventions ("no JavaScript," "mobile-first CSS"), and the exact Terraform/deployment commands for this project. The test that mattered: asking Claude to "add a React component to the homepage" — it refused, correctly citing the no-JavaScript convention from CLAUDE.md, and offered a CSS-only alternative instead. That's the difference between a model guessing and a model that actually knows the project's rules.

## Skills: repeatable, checklist-driven commands

A Skill packages a fixed procedure behind a slash command so the same check runs the same way every time, regardless of how the request is phrased. I designed and used a `/tf-plan` skill — YAML frontmatter defining allowed tools and model, then explicit instructions to run `terraform plan`, parse resource counts, flag high-risk changes (IAM, S3 policy edits), and produce a structured risk summary, with a hard constraint that it can never run `terraform apply`. Running the full scaffold → plan → apply → deploy pipeline through chained skill commands, with `terraform init` as the one manual step between them, took a multi-step deployment down to a handful of commands.

## Subagents: a specialized AI team with real access boundaries

Where Skills reuse instructions in the current conversation, Subagents are isolated delegations — no shared history, their own tool access, their own model choice. I built three with deliberately different privilege levels:

| Subagent | Tools | Model | Why |
|---|---|---|---|
| `cost-optimizer` | Read, Grep, Glob (read-only) | Haiku | Cost review is pattern-matching against known configs — fast and cheap is the right trade |
| `security-auditor` | Read, Grep, Glob (read-only) | Sonnet | Judging least-privilege IAM or OIDC scoping needs real reasoning, and an auditor must never be able to modify what it's reviewing |
| `tf-writer` | Read, Write, Edit, Glob, Grep | inherit | Its entire job is producing and modifying files — write access isn't optional, it's the point |

Running `Audit my Terraform files for security issues` correctly delegated to the security-auditor, which returned a severity-ranked report (one CRITICAL, one HIGH, one MEDIUM finding) rather than a vague summary — the structured-output format is what makes it usable by a human or another tool downstream.

## MCP: connecting to real infrastructure state

An MCP server is what lets Claude Code act on your *actual* environment instead of reasoning from stale training data — I wired up Terraform and AWS MCP servers over stdio, with credentials isolated in `.claude/settings.local.json` (gitignored) rather than the shareable, committed `.mcp.json`. The full loop I validated: the security-auditor subagent flags missing S3 server-side encryption → the `tf-writer` subagent uses the Terraform MCP server to read the existing resource and write the fix → re-running the auditor confirms the finding is gone. Audit, fix, and verify, without touching the Terraform files by hand at any point.

## Hooks: the safety layer underneath all of it

Agentic tooling that can run real commands against real infrastructure needs interception points, not just good intentions. I wired three hook scripts into `settings.json`: `UserPromptSubmit` (scans the raw prompt text for red flags before Claude even processes it — the **SAY** gate), `PreToolUse` matched on Bash (inspects the actual command about to execute — the **DO** gate), and `PostToolUse` (logs what ran, after the fact — the **LOG** trail). The reason both a prompt-level and a tool-level guard matter: "clean up the old staging environment" never says *delete*, but could resolve to `terraform destroy`. A prompt scanner misses it because the dangerous word was never typed; the tool-level hook catches it because it inspects the literal command Claude is about to run, not the human-readable intent behind it.

## What I learned

- **Skills and Subagents solve different problems.** If the task needs the conversation's existing context, it's a Skill. If it should work identically for anyone, cold, it's a Subagent.
- **Read-only-by-default is the right posture for anything that reviews infrastructure.** Write access should be scoped to the one agent whose entire job is writing.
- **A single prompt-level guard is not a security boundary.** Real safety comes from checking the actual command about to execute, not just the words used to ask for it.
