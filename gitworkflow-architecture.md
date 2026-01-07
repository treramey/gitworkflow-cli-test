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
│     git push -u origin jd/add-user-auth                                         │
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
│  ┌─────────────────┐  ┌─────────────────┐                                      │
│  │   GATE 1        │  │   GATE 2        │                                      │
│  │   CI Tests      │  │   Human Review  │                                      │
│  │                 │  │                 │                                      │
│  │   ✓ Type check  │  │   ✓ Architecture│                                      │
│  │   ✓ Build       │  │   ✓ Business    │                                      │
│  │                 │  │     logic       │                                      │
│  │   [Automated]   │  │   ✓ Approval    │                                      │
│  │   [Blocking]    │  │                 │                                      │
│  │                 │  │   [Manual]      │                                      │
│  │   On pass:      │  │   [Required for │                                      │
│  │   "env: dev"    │  │    merge only]  │                                      │
│  │   auto-added    │  │                 │                                      │
│  └────────┬────────┘  └────────┬────────┘                                      │
│           │                    │                                               │
│           └────────────────────┘                                               │
│                     │                                                          │
│                     ▼                                                          │
│         ┌───────────────────────┐                                              │
│         │  CI PASS              │                                              │
│         │                       │                                              │
│         │  Label added:         │                                              │
│         │  "env: dev"           │                                              │
│         │  (automatic)          │                                              │
│         └───────────────────────┘                                              │
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
│  CI pass         │        │  Manual label    │        │  PR merge        │
│  (auto label)    │        │  "env: staging"  │        │                  │
│                  │        │                  │        │                  │
│  Includes:       │        │  Includes:       │        │  Includes:       │
│  Topics with     │        │  Topics with     │        │  Graduated       │
│  "env: dev" +    │        │  "env: dev" +    │        │  topics          │
│  "status:        │        │  "env: staging"  │        │                  │
│   ci-passed"     │        │  + "status:      │        │                  │
│                  │        │   ci-passed"     │        │                  │
│                  │        │                  │        │                  │
│  Rebuilt:        │        │  Rebuilt:        │        │  Never           │
│  On label add +  │        │  Daily 6am UTC + │        │  rebased         │
│  on push +       │        │  manual dispatch │        │                  │
│  on PR close     │        │                  │        │                  │
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
│  add staging     │        │  ready to ship"  │        │ - Changelog      │
│  label"          │        │                  │        │ - Git tag        │
│                  │        │ Get PR approval  │        │ - GitHub Release │
│ User adds        │        │ Merge PR to      │        │ - Notifications  │
│ "env: staging"   │        │ master           │        │                  │
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
                      create  │  push  │  merged to master
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
  env: dev                 │ CI passed, enters dev            │ CI workflow (auto)
  status: conflict         │ Merge conflict in dev rebuild    │ Rebuild workflow
  env: staging             │ Promoted to staging              │ Manual (user)
  blocked                  │ Do not integrate                 │ Manual

  Merge Gate (rebuild-gate):
  - Required to merge to master
  - Requires: CI passed + env: dev + env: staging + PR approval


┌─────────────────────────────────────────────────────────────────────────────────┐
│                           WORKFLOW FILES                                        │
└─────────────────────────────────────────────────────────────────────────────────┘

  .github/
  └── workflows/
      ├── pr-checks.yml              # CI + Label management + Merge gate
      ├── rebuild-and-deploy.yml     # Rebuild dev/staging + deploy to servers
      ├── hotfix.yml                 # Hotfix release + merge maint → master
      └── release.yml                # Triggered on PR merge to master


┌─────────────────────────────────────────────────────────────────────────────────┐
│                         REBUILD TRIGGERS                                        │
└─────────────────────────────────────────────────────────────────────────────────┘

  The `rebuild-and-deploy.yml` workflow triggers on:

  DEV REBUILD:
  1. **Label Event**: When "env: dev" label is added to a PR
  2. **Push Event**: When code is pushed to a PR that has "env: dev" + "status: ci-passed" labels
  3. **PR Closed**: To remove merged/closed topics from dev
  4. **Manual**: workflow_dispatch with rebuild_dev=true

  STAGING REBUILD:
  1. **Schedule**: Daily at 6am UTC
  2. **Manual**: workflow_dispatch with rebuild_staging=true

  Staging Requirements:
  - Topic must have ALL THREE labels: "env: dev" + "env: staging" + "status: ci-passed"
  - Topics without all three are excluded from staging rebuild

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
  │ ✅ Deployed to    │                             │ 2. Labels REMOVED:│
  │    Alpha          │                             │    "env: dev"     │
  │ ✅ Comment posted │                             │    "env: staging" │
  │    on PR          │                             │                   │
  └───────────────────┘                             │ 3. Label ADDED:   │
                                                    │   "status:conflict│
                                                    │ 4. Comment posted │
                                                    │    with fix       │
                                                    │    instructions   │
                                                    │ 5. NOT deployed   │
                                                    └─────────┬─────────┘
                                                              │
                                                              ▼
                                                    ┌───────────────────┐
                                                    │ Developer rebases │
                                                    │ and pushes        │
                                                    │                   │
                                                    │ git rebase        │
                                                    │   origin/master   │
                                                    │ git push -f       │
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
                                                    │   "env: dev"      │
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
  │  ## Merge Conflict in Integration Branch                                    │
  │                                                                             │
  │  Your branch `ab/refactor-login` could not be merged into `dev` due to a     │
  │  merge conflict.                                                            │
  │                                                                             │
  │  **What this means:**                                                       │
  │  - Your code is NOT currently deployed to the Alpha server                  │
  │  - The `env: dev` and `env: staging` labels have been removed               │
  │  - You need to resolve the conflict before your code can be integrated      │
  │                                                                             │
  │  **How to fix:**                                                            │
  │                                                                             │
  │  ```bash                                                                    │
  │  git checkout ab/refactor-login                                             │
  │  git fetch origin                                                           │
  │  git rebase origin/master                                                   │
  │  # Resolve conflicts...                                                     │
  │  git push --force-with-lease                                                │
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

  3. QUALITY GATES
     - CI runs on PR, must pass before dev integration
     - Human review required before merge to master (not before dev)
     - Staging validates before production

  4. SINGLE PR WORKFLOW
     - One PR (to master) handles the entire journey
     - Labels track progress through the pipeline
     - PR is merged only when ready for production

   5. CLEAR PROMOTION PATH
       - topic → dev (automatic, after CI pass, "env: dev" label auto-added)
       - dev → staging (manual, user adds "env: staging" label, daily rebuild)
       - staging → master (PR merge after all gates pass)
       
       Merge to master requires all four:
       - CI passed (status: ci-passed)
       - In dev (env: dev label)
       - In staging (env: staging label)
       - PR approval

  6. CONFLICT HANDLING
     - Conflicts during dev rebuild are detected and reported
     - Both env labels removed, developer notified via PR comment
     - Developer rebases, CI re-runs, label re-added
     - Next rebuild includes the fixed topic
```
