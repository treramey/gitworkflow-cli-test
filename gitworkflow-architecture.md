# Gitworkflow CI/CD Architecture

## Overview

This document describes a gitworkflow-based CI/CD pipeline with quality gates, integration branches, and automated deployments to multiple environments.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  DEVELOPER                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  1. CREATE FEATURE BRANCH                                                       │
│                                                                                 │
│     git checkout origin/master -b <initials>/<feature-name>                     │
│     Example: jd/add-user-auth                                                   │
│                                                                                 │
│     Convention: Always branch from master, never from staging or dev                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  2. DEVELOP & PUSH                                                              │
│                                                                                 │
│     git commit -m "feat: add user authentication"                               │
│     git devsh -u origin jd/add-user-auth                                         │
│                                                                                 │
│     Convention: Use conventional commits (feat:, fix:, docs:, etc.)             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  3. OPEN PULL REQUEST TO MASTER                                                 │
│                                                                                 │
│     gh pr create --base master --title "feat: Add user authentication"          │
│                                                                                 │
│     Note: Single PR handles entire journey from development to production       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              QUALITY GATES                                      │
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │   GATE 1        │  │   GATE 2        │  │   GATE 3        │                 │
│  │   CI Tests      │  │   AI Review     │  │   Human Review  │                 │
│  │                 │  │                 │  │                 │                 │
│  │   ✓ Lint        │  │   ✓ Code        │  │   ✓ Architecture│                 │
│  │   ✓ Format      │  │     quality     │  │   ✓ Business    │                 │
│  │   ✓ Unit tests  │  │   ✓ Security    │  │     logic       │                 │
│  │   ✓ Type check  │  │   ✓ Best        │  │   ✓ Approval    │                 │
│  │   ✓ Build       │  │     practices   │  │                 │                 │
│  │   ✓ Security    │  │   ✓ Bug         │  │                 │                 │
│  │     audit       │  │     detection   │  │                 │                 │
│  │                 │  │                 │  │                 │                 │
│  │   [Automated]   │  │   [Automated]   │  │   [Manual]      │                 │
│  │   [Blocking]    │  │   [Advisory]    │  │   [Blocking]    │                 │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                 │
│           │                    │                    │                          │
│           └────────────────────┴────────────────────┘                          │
│                                │                                               │
│                                ▼                                               │
│                    ┌───────────────────────┐                                   │
│                    │  ALL GATES PASS       │                                   │
│                    │                       │                                   │
│                    │  Label added:         │                                   │
│                    │  "status: ready"      │                                   │
│                    │                       │                                   │
│                    └───────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         INTEGRATION BRANCHES                                    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
          ▼                            ▼                            ▼
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│       dev         │        │      staging        │        │     master       │
│  (Proposed       │        │  (Stabilization) │        │    (Release)     │
│   Updates)       │        │                  │        │                  │
│                  │        │                  │        │                  │
│  ┌────────────┐  │        │  ┌────────────┐  │        │  ┌────────────┐  │
│  │   Alpha    │  │        │  │  Staging   │  │        │  │ Production │  │
│  │   Server   │  │        │  │  Server    │  │        │  │  Server    │  │
│  │            │  │        │  │            │  │        │  │            │  │
│  │ alpha.     │  │        │  │ staging.   │  │        │  │ app.com    │  │
│  │ app.com    │  │        │  │ app.com    │  │        │  │            │  │
│  └────────────┘  │        │  └────────────┘  │        │  └────────────┘  │
│                  │        │                  │        │                  │
│  Entry:          │        │  Entry:          │        │  Entry:          │
│  Automatic       │        │  Release mgr     │        │  PR merge        │
│  (rebuild)       │        │  promotes        │        │                  │
│                  │        │                  │        │                  │
│  Includes:       │        │  Includes:       │        │  Includes:       │
│  Topics with     │        │  Topics ready    │        │  Graduated       │
│  "status: ready" │        │  for release     │        │  topics          │
│  label           │        │                  │        │                  │
│                  │        │                  │        │                  │
│                  │        │                  │        │                  │
│  Rebuilt:        │        │  Rebuilt:        │        │  Never           │
│  On label add +  │        │  After releases  │        │  rebased         │
│  every 2 hours   │        │  (or on-demand)  │        │                  │
│                  │        │                  │        │                  │
│  Audience:       │        │  Audience:       │        │  Audience:       │
│  - Developers    │        │  - QA team       │        │  - Customers     │
│  - Early testers │        │  - Stakeholders  │        │  - End users     │
│                  │        │  - Beta users    │        │                  │
└────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
         │                           │                           │
         │      ┌────────────────────┘                           │
         │      │                                                │
         ▼      ▼                                                ▼
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│ Alpha Testing    │        │ Staging Testing  │        │ Release          │
│                  │  ───►  │                  │  ───►  │                  │
│ "Works well,     │        │ "QA approved,    │        │ - Version bump   │
│  promote it"     │        │  ready to ship"  │        │ - Changelog      │
│                  │        │                  │        │ - Git tag        │
│ Release mgr      │        │ Merge PR to      │        │ - GitHub Release │
│ promotes to staging │        │ master           │        │ - Notifications  │
└──────────────────┘        └──────────────────┘        └──────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                              BRANCH TOPOLOGY                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

  master ────●─────────────────────────────────────────●───────────● (v1.2.0)
              \                                       /           /
               \                                     /           /
  maint ────────●─────────────────────────────────────────────────● (hotfixes)
                 \                                 /
                  \                               /
  staging ────────────●────●────●───────────────────●──────────────── (rebuilt)
                    \       \                   /
                     \       \                 /
  dev ─────────────────●───────●────●────●─────●──────────────────── (rebuilt)
                       \           \         /
                        \           \       /
  jd/feature             ●────●────●───●───●  (topic branch)
                         ↑    ↑    ↑   ↑   ↑
                      create  │  devsh  │  merged to master
                              │        │
                           merged    fixes
                           to dev


┌─────────────────────────────────────────────────────────────────────────────────┐
│                              TOPIC STATUS                                       │
└─────────────────────────────────────────────────────────────────────────────────┘

  Symbol │ Branch  │ Meaning                    │ Server
  ───────┼─────────┼────────────────────────────┼─────────────
    .    │ (none)  │ Not integrated yet         │ None
    -    │ dev      │ Proposed, alpha testing    │ Alpha
    +    │ staging    │ Stabilizing, beta testing  │ Staging
    *    │ maint   │ In maintenance release     │ Production
    =    │ master  │ Graduated, released        │ Production

  View status: git where


┌─────────────────────────────────────────────────────────────────────────────────┐
│                              LABELS                                             │
└─────────────────────────────────────────────────────────────────────────────────┘

  Label                    │ Meaning                          │ Added By
  ─────────────────────────┼──────────────────────────────────┼──────────────
  status: ci-passed        │ Automated tests passed           │ CI workflow
  status: ci-failed        │ Automated tests failed           │ CI workflow
  status: needs-review     │ Waiting for human approval       │ CI workflow
  status: ready            │ CI + approval, enters dev        │ CI workflow
  status: conflict         │ Merge conflict in dev rebuild    │ Rebuild workflow
  env: staging             │ Promoted to staging              │ Promote workflow
  env: qa                  │ Ready to merge to master         │ CI workflow
  blocked                  │ Do not integrate                 │ Manual

  Merge Gate:
  - "Merge Gate" check is REQUIRED to merge to master
  - Only passes when "env: qa" label is present
  - Requires: CI passed + code review approval + env: staging label


┌─────────────────────────────────────────────────────────────────────────────────┐
│                           WORKFLOW FILES                                        │
└─────────────────────────────────────────────────────────────────────────────────┘

  .github/
  └── workflows/
      ├── pr-checks.yml              # CI + AI Review + Label management
      ├── rebuild-and-deploy.yml     # Rebuild dev/staging + deploy to servers
      ├── promote-to-staging.yml     # Release manager promotes topic
      └── release.yml                # Triggered on PR merge to master


┌─────────────────────────────────────────────────────────────────────────────────┐
│                         REBUILD TRIGGERS                                        │
└─────────────────────────────────────────────────────────────────────────────────┘

  The `rebuild-and-deploy.yml` workflow triggers on:

  1. **Label Event** (instant): When a user manually adds the `status: ready`
     label to a PR, the workflow triggers immediately.
     
     Note: Labels added by workflows (e.g., pr-checks adding the label after
     approval) do NOT trigger the rebuild - this is a GitHub limitation.
     For these cases, use manual trigger or wait for the scheduled run.

  2. **Schedule** (every 2 hours): Safety net to catch any missed events,
     conflict re-checks, and workflow-added labels.

  3. **Manual** (workflow_dispatch): On-demand via GitHub Actions UI or `gh` CLI.

  Concurrency:
  - Uses `concurrency: { group: rebuild-dev, cancel-in-progress: false }`
  - Multiple triggers queue up rather than cancelling each other
  - Ensures all labeled PRs get processed


┌─────────────────────────────────────────────────────────────────────────────────┐
│                         CONFLICT HANDLING                                       │
└─────────────────────────────────────────────────────────────────────────────────┘

  When a topic branch cannot merge cleanly into `dev` during rebuild:

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                          dev REBUILD RUNS                                    │
  └─────────────────────────────────────────────────────────────────────────────┘
                                    │
          ┌─────────────────────────┴─────────────────────────┐
          │                                                   │
          ▼                                                   ▼
  ┌───────────────────┐                             ┌───────────────────┐
  │ jd/add-user-auth  │                             │ ab/refactor-login │
  │                   │                             │                   │
  │ Merges cleanly ✅ │                             │ CONFLICT! ❌      │
  └───────────────────┘                             └─────────┬─────────┘
          │                                                   │
          ▼                                                   ▼
  ┌───────────────────┐                             ┌───────────────────┐
  │ ✅ In dev          │                             │ 1. Merge aborted  │
  │ ✅ Deployed to    │                             │ 2. Label REMOVED: │
  │    Alpha          │                             │    "status: ready"│
  │ ✅ Comment posted │                             │                   │
  │    on PR          │                             │ 3. Label ADDED:   │
  └───────────────────┘                             │   "status:conflict│
                                                    │ 4. Comment posted │
                                                    │    with fix       │
                                                    │    instructions   │
                                                    │ 5. NOT deployed   │
                                                    └─────────┬─────────┘
                                                              │
                                                              ▼
                                                    ┌───────────────────┐
                                                    │ Developer rebases │
                                                    │ and devshes        │
                                                    │                   │
                                                    │ git rebase        │
                                                    │   origin/master   │
                                                    │ git devsh -f       │
                                                    └─────────┬─────────┘
                                                              │
                                                              ▼
                                                    ┌───────────────────┐
                                                    │ CI re-runs        │
                                                    │                   │
                                                    │ ✅ Passes →       │
                                                    │   Labels updated  │
                                                    │   "status:        │
                                                    │    conflict"      │
                                                    │   removed         │
                                                    │   "status: ready" │
                                                    │   re-added        │
                                                    └─────────┬─────────┘
                                                              │
                                                              ▼
                                                    ┌───────────────────┐
                                                    │ Next dev rebuild   │
                                                    │                   │
                                                    │ Topic merges      │
                                                    │ successfully ✅   │
                                                    │                   │
                                                    │ Deployed to Alpha │
                                                    └───────────────────┘

  CONFLICT NOTIFICATION (Posted to PR):

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  ## ⚠️ Merge Conflict in Integration Branch                                 │
  │                                                                             │
  │  Your branch `ab/refactor-login` could not be merged into `dev` due to a     │
  │  merge conflict.                                                            │
  │                                                                             │
  │  **What this means:**                                                       │
  │  - Your code is NOT currently deployed to the Alpha server                  │
  │  - The `status: ready` label has been removed                               │
  │  - You need to resolve the conflict before your code can be integrated      │
  │                                                                             │
  │  **How to fix:**                                                            │
  │                                                                             │
  │  ```bash                                                                    │
  │  git checkout ab/refactor-login                                             │
  │  git fetch origin                                                           │
  │  git rebase origin/master                                                   │
  │  # Resolve conflicts...                                                     │
  │  git devsh --force-with-lease                                                │
  │  ```                                                                        │
  │                                                                             │
  │  **Conflicting with:** `jd/add-user-auth`                                   │
  └─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                           KEY PRINCIPLES                                        │
└─────────────────────────────────────────────────────────────────────────────────┘

  1. TOPICS ARE FIRST-CLASS CITIZENS
     - All development happens on topic branches
     - Topics maintain their identity through the entire pipeline
     - Topic history is preserved via merge commits

  2. INTEGRATION BRANCHES ARE EPHEMERAL
     - dev and staging are rebuilt regularly (rewind and rebuild)
     - Never commit directly to dev or staging
     - Only master and maint are permanent

  3. QUALITY GATES BEFORE INTEGRATION
     - CI, AI review, and human review happen on the PR
     - Only approved code enters dev
     - Servers always have tested code

  4. SINGLE PR WORKFLOW
     - One PR (to master) handles the entire journey
     - Labels track progress through the pipeline
     - PR is merged only when ready for production

   5. CLEAR PROMOTION PATH
       - topic → dev (automatic, after gates pass, "status: ready" label)
       - dev → staging (manual, release manager promotes, "env: staging" label)
       - staging → master (PR merge after "env: qa" label, triggers release)
       
       Merge to master requires all three:
       - CI passed
       - Code review approval  
       - env: staging label (proves staging validation)

  6. CONFLICT HANDLING
     - Conflicts during dev rebuild are detected and reported
     - Label removed, developer notified via PR comment
     - Developer rebases, CI re-runs, label re-added
     - Next rebuild includes the fixed topic
```
