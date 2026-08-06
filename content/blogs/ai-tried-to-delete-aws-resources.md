---
title: "My AI Tried to Delete All My AWS Resources. A Hook Stopped It."
date: 2026-03-18
description: "Why a prompt-level guard isn't enough, and how a tool-level hook caught a destructive command before it ran."
tags:
  - "Claude Code"
  - "Agentic AI"
  - "AWS"
toc: false
---

A prompt-level guard can be talked around — indirect phrasing slips past it every time. What actually stopped a destructive command from reaching my AWS account was a hook that inspected the literal command Claude was about to run, not the words used to ask for it. This is the story of watching that safety layer do its job.

Read the full story on [Medium](https://medium.com/@afomaegbuonu/my-ai-tried-to-delete-all-my-aws-resources-a-hook-stopped-it-3a317cbaacad).

For the fuller technical write-up, see [Agentic DevOps with Claude Code]({{< relref "agentic-devops-claude-code.md" >}}) in my case studies.
