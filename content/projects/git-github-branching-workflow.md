---
title: "Git & GitHub Branching Workflow"
date: 2026-02-20
description: "CodeTrack: a from-scratch Git workflow covering local vs global identity, feature branching, and a real fork-based pull request."
badges:
  - "Git"
  - "GitHub"
  - "CI/CD"
links:
  - icon: fab fa-github
    url: "https://github.com/AfomaCloud21/devops-micro-internship-pravinmishra/tree/main/week-04-git-and-github"
tags:
  - "Git"
  - "GitHub"
  - "Version Control"
toc: true
showInHome: false
---

Most DevOps failures I ran into later in this portfolio trace back to version control discipline — a bad merge, an untracked change, a missing branch protection rule. This project (codename **CodeTrack**) was about building that discipline deliberately, from `git init` through a real open-source-style pull request.

## Identity and initialization

I started by separating **local** from **global** Git identity: configuring `user.name`/`user.email` scoped only to the CodeTrack repo (`git config --local`), and a separate global default for every other repo on the machine. Git resolves configuration in the order local → global → system, and getting that order right matters the moment you're juggling personal and work repos on the same machine.

## Tracking, staging, and a clean commit history

With `index.html` and `style.css` scaffolded, I practiced the staging discipline that keeps history readable: `git status` to see what's untracked, `git add` deliberately rather than blanket-adding, and small, single-purpose commits with descriptive messages — one commit for the initial UI, a separate commit for a later content change, verified with `git log --oneline`. The site was then deployed to an EC2 instance running Nginx, the same manual-deploy pattern used across this portfolio, specifically to make the value of automated CI/CD tangible by first feeling the manual version.

## Branching workflow: isolating a feature

To practice GitHub Flow properly, I added a Contact page entirely on a feature branch:

1. Confirmed a clean starting point on the default branch
2. Created and switched to `feature/contact-page`
3. Added `contact.html` and committed it as an atomic, reviewable change
4. Added the navigation link to it in a **separate** commit, keeping concerns isolated
5. Verified isolation — switching back to `main` showed neither the file nor the link existed there
6. Merged the feature branch back into `main` and validated the merged result in the browser
7. Reviewed the merge in `git log`'s graph view, then cleaned up the merged branch

## A real fork-based pull request

The second half of this project moved from solo workflow to open-source collaboration mechanics, forking `pravinmishraaws/devops-micro-internship-interviews` rather than working directly against the upstream repo:

- Authenticated over HTTPS using a Personal Access Token (GitHub no longer accepts password auth for Git operations)
- Cloned **my fork**, not the upstream, and configured two remotes — `origin` (my fork) and `upstream` (the original) — so I could pull updates without ever pushing directly to the source project
- Created a dedicated `feature-readme-update` branch, made a single focused change, and committed it
- Synced with upstream before pushing: fetched, merged into local `main`, then rebased the feature branch on top, keeping the PR clean
- Opened the pull request manually after GitHub's "Compare & pull request" prompt didn't appear — carefully setting the base (upstream/main) and head (my fork's feature branch) by hand

## What I learned

- **Local vs. global config prevents identity mistakes** that are easy to make and awkward to unwind in shared repos.
- **Atomic commits and isolated feature branches** aren't process for its own sake — they're what make a diff reviewable and a rollback safe.
- **Forking and remote separation is the actual mechanism** behind every open-source contribution workflow, not just a GitHub UI button.
