# TASKS.md - Gitworkflow Implementation

## Project Overview

Implement a gitworkflow-based CI/CD pipeline with quality gates, integration branches (`dev`, `staging`, `master`), and automated deployments to Alpha, Staging, and Production environments.

---

## Phase 1: Repository Setup

### 1.1 Branch Structure
- [ ] Create `master` branch (if not exists)
- [ ] Create `maint` branch from `master`
- [ ] Create `staging` branch from `master`
- [ ] Create `dev` branch from `master`
- [ ] Configure branch protection rules:
  - [ ] `master`: Require PR, require status checks, no force devsh
  - [ ] `maint`: Require PR, no force devsh
  - [ ] `staging`: Allow force devsh from bot only
  - [ ] `dev`: Allow force devsh from bot only

### 1.2 Labels
- [ ] Create label: `ci-passed` (green)
- [ ] Create label: `ci-failed` (red)
- [ ] Create label: `needs-review` (yellow)
- [ ] Create label: `ready-for-integration` (devrple)
- [ ] Create label: `has-conflict` (orange)
- [ ] Create label: `in-staging` (blue)
- [ ] Create label: `blocked` (black)

### 1.3 Secrets & Variables
- [ ] Create `GH_PAT` secret (Personal Access Token with repo permissions)
- [ ] Create `ANTHROPIC_API_KEY` secret (for AI code review)
- [ ] Configure deployment secrets:
  - [ ] Alpha server credentials
  - [ ] Staging server credentials
  - [ ] Production server credentials

### 1.4 Environments
- [ ] Create `alpha` environment
  - [ ] Set URL: `https://alpha.yourapp.com`
  - [ ] No protection rules (auto-deploy)
- [ ] Create `staging` environment
  - [ ] Set URL: `https://staging.yourapp.com`
  - [ ] Optional: Add required reviewers
- [ ] Create `production` environment
  - [ ] Set URL: `https://yourapp.com`
  - [ ] Add required reviewers
  - [ ] Add deployment branch rule: `master` only

---

## Phase 2: CI Workflows

### 2.1 PR Checks Workflow (`.github/workflows/pr-checks.yml`)
- [ ] Create workflow file
- [ ] Implement CI job:
  - [ ] Checkout code
  - [ ] Setup Node.js (or your language)
  - [ ] Install dependencies
  - [ ] Run linter
  - [ ] Run formatter check
  - [ ] Run type checker
  - [ ] Run unit tests
  - [ ] Run build
  - [ ] Run security audit
- [ ] Implement AI review job:
  - [ ] Get changed files
  - [ ] Call Claude API for review
  - [ ] Post review as PR comment
- [ ] Implement gate check job:
  - [ ] Check CI status
  - [ ] Check human review status
  - [ ] Update labels based on status
  - [ ] Remove `has-conflict` label when CI passes (conflict resolved)
- [ ] Test workflow:
  - [ ] Open test PR
  - [ ] Verify CI runs
  - [ ] Verify AI review posts comment
  - [ ] Verify labels are applied correctly

### 2.2 Topic Branch CI (Optional)
- [ ] Create `.github/workflows/topic-ci.yml`
- [ ] Run CI on devsh to topic branches (`[a-z][a-z]/**`)
- [ ] Set commit status for branch

---

## Phase 3: Integration Branch Workflows

### 3.0 Conflict Handling Strategy
The rebuild workflow must handle merge conflicts gracefully:

**When a conflict occurs during `dev` rebuild:**
1. Abort the merge (`git merge --abort`)
2. Remove `ready-for-integration` label from the PR
3. Add `has-conflict` label to the PR
4. Post a comment explaining:
   - What happened
   - That the code is NOT deployed
   - How to fix (rebase instructions)
   - Which topic it conflicted with
5. Continue rebuilding with remaining topics
6. Track and report all conflicts in workflow summary

**When developer fixes the conflict:**
1. Developer rebases on `origin/master`
2. Developer devshes (force devsh since rebased)
3. CI re-runs on the PR
4. If CI passes:
   - Remove `has-conflict` label
   - Re-add `ready-for-integration` label
5. Next `dev` rebuild will include the topic

**Git rerere:**
- Enable `rerere.enabled = true` in workflows
- Remembers conflict resolutions
- Can auto-resolve previously seen conflicts

### 3.1 Rebuild & Deploy Workflow (`.github/workflows/rebuild-and-deploy.yml`)
- [ ] Create workflow file
- [ ] Implement scheduled trigger (every 4 hours)
- [ ] Implement manual trigger with indevts
- [ ] Implement `rebuild-dev` job:
  - [ ] Fetch topics with `ready-for-integration` label
  - [ ] Checkout and reset `dev` to `master`
  - [ ] Phase 1: Merge topics from `staging`
  - [ ] Create marker commit `### Match 'staging'`
  - [ ] Phase 2: Merge labeled topics
  - [ ] Track merged vs conflicted topics
  - [ ] Force devsh `dev`
- [ ] Implement conflict handling:
  - [ ] Detect merge conflicts during rebuild
  - [ ] Remove `ready-for-integration` label from conflicted PRs
  - [ ] Add `has-conflict` label to conflicted PRs
  - [ ] Post comment with conflict details and fix instructions
  - [ ] Include which topic it conflicted with
- [ ] Implement `deploy-alpha` job:
  - [ ] Checkout `dev` branch
  - [ ] Build application
  - [ ] Deploy to Alpha server
  - [ ] Post deployment comments on PRs
- [ ] Implement `rebuild-staging` job:
  - [ ] Get topics currently in `staging`
  - [ ] Checkout and reset `staging` to `master`
  - [ ] Re-merge non-graduated topics
  - [ ] Force devsh `staging`
- [ ] Implement `deploy-staging` job:
  - [ ] Checkout `staging` branch
  - [ ] Build application
  - [ ] Deploy to Staging server
- [ ] Test workflow:
  - [ ] Trigger manually
  - [ ] Verify `dev` is rebuilt correctly
  - [ ] Verify Alpha deployment
  - [ ] Verify `staging` is rebuilt correctly
  - [ ] Verify Staging deployment

### 3.2 Promote to Next Workflow (`.github/workflows/promote-to-staging.yml`)
- [ ] Create workflow file
- [ ] Implement manual trigger with topic branch indevt
- [ ] Validate topic exists
- [ ] Check if topic is in `dev`
- [ ] Merge topic to `staging`
- [ ] Update PR label to `in-staging`
- [ ] Post comment on PR
- [ ] Test workflow:
  - [ ] Promote a test topic
  - [ ] Verify merge to `staging`
  - [ ] Verify label update

---

## Phase 4: Release Workflow

### 4.1 Release Workflow (`.github/workflows/release.yml`)
- [ ] Create workflow file
- [ ] Trigger on PR merge to `master`
- [ ] Implement release job:
  - [ ] Check and merge `maint` if needed
  - [ ] Determine version bump from conventional commits
  - [ ] Bump version using `bumpp`
  - [ ] Generate changelog
  - [ ] Archive old `maint` branch
  - [ ] Update `maint` to match `master`
  - [ ] Push `master` with tags
  - [ ] Rebuild `staging` (remove graduated topics)
  - [ ] Rebuild `dev` (remove graduated topics)
  - [ ] Create GitHub Release
- [ ] Implement deploy-production job:
  - [ ] Checkout `master`
  - [ ] Build application
  - [ ] Deploy to Production
- [ ] Test workflow:
  - [ ] Merge test PR to `master`
  - [ ] Verify version bump
  - [ ] Verify changelog
  - [ ] Verify GitHub Release
  - [ ] Verify Production deployment

---

## Phase 5: Developer Tooling

### 5.1 Git Aliases
- [ ] Create `.gitconfig` snippet for team
- [ ] Document aliases:
  - [ ] `git lg` / `git lgp` - Pretty log
  - [ ] `git topics` - List topic branches
  - [ ] `git where` - Show topic status
  - [ ] `git topiclg <branch>` - Topic history
  - [ ] `git ready-to-graduate` - Topics ready for release
  - [ ] `git release-check` - Pre-release checklist
- [ ] Add to team onboarding docs

### 5.2 Local Scripts
- [ ] Create `scripts/gitworkflow-rebuild` script
- [ ] Implement `status` command
- [ ] Implement `staging` rebuild command
- [ ] Implement `dev` rebuild command
- [ ] Implement `--dry-run` mode
- [ ] Add to `package.json` scripts (optional)

### 5.3 Documentation
- [ ] Create `docs/GITWORKFLOW.md` - Workflow explanation
- [ ] Create `docs/DEVELOPER_GUIDE.md` - Day-to-day usage
- [ ] Create `docs/RELEASE_MANAGER_GUIDE.md` - Promotion & release
- [ ] Update `CONTRIBUTING.md` with branching conventions

---

## Phase 6: Deployment Infrastructure

### 6.1 Alpha Server
- [ ] Provision Alpha server/environment
- [ ] Configure domain: `alpha.yourapp.com`
- [ ] Setup deployment method:
  - [ ] Option A: SSH + rsync
  - [ ] Option B: Docker registry + devll
  - [ ] Option C: Vercel/Netlify preview
  - [ ] Option D: Kubernetes namespace
- [ ] Configure environment variables
- [ ] Test deployment manually

### 6.2 Staging Server
- [ ] Provision Staging server/environment
- [ ] Configure domain: `staging.yourapp.com`
- [ ] Setup deployment method (same as Alpha)
- [ ] Configure environment variables
- [ ] Test deployment manually

### 6.3 Production Server
- [ ] Verify Production deployment method
- [ ] Configure rollback procedure
- [ ] Setup monitoring/alerting
- [ ] Document deployment runbook

---

## Phase 7: Testing & Validation

### 7.1 End-to-End Test
- [ ] Create test feature branch: `test/e2e-validation`
- [ ] Push and open PR to `master`
- [ ] Verify CI runs and passes
- [ ] Verify AI review comment
- [ ] Request and receive human review
- [ ] Verify `ready-for-integration` label added
- [ ] Trigger `dev` rebuild
- [ ] Verify topic in `dev`
- [ ] Verify Alpha deployment
- [ ] Test on Alpha server
- [ ] Promote to `staging`
- [ ] Verify Staging deployment
- [ ] Test on Staging server
- [ ] Merge PR to `master`
- [ ] Verify release created
- [ ] Verify Production deployment
- [ ] Verify topic removed from `staging` and `dev`
- [ ] Clean up test branch

### 7.2 Edge Cases
- [ ] Test: PR with failing CI (should not get label)
- [ ] Test: Push fix to branch in `dev` (should update on rebuild)
- [ ] Test: Merge conflict in `dev` rebuild (should skip topic, remove label, notify)
- [ ] Test: Conflict resolution flow (rebase, devsh, CI re-runs, label re-added)
- [ ] Test: Two topics conflicting with each other
- [ ] Test: Topic depends on another topic
- [ ] Test: Hotfix via `maint` branch
- [ ] Test: Concurrent PRs merged to `master`
- [ ] Test: rerere auto-resolution of previously seen conflict

---

## Phase 8: Team Onboarding

### 8.1 Documentation
- [ ] Write quick-start guide
- [ ] Create video walkthrough (optional)
- [ ] Document common scenarios:
  - [ ] "How do I start a new feature?"
  - [ ] "How do I fix a bug found in staging?"
  - [ ] "How do I handle a merge conflict?"
  - [ ] "How do I resolve a conflict in dev rebuild?"
  - [ ] "How do I do a hotfix?"
  - [ ] "What do the different labels mean?"

### 8.2 Training
- [ ] Schedule team walkthrough session
- [ ] Pair with developers on first features
- [ ] Collect feedback and iterate

### 8.3 Rollout
- [ ] Start with pilot team/project
- [ ] Monitor for issues
- [ ] Expand to full team
- [ ] Establish office hours for questions

---

## Checklist Summary

### Minimum Viable Implementation
- [ ] Branch structure (`master`, `maint`, `staging`, `dev`)
- [ ] Labels created
- [ ] `pr-checks.yml` workflow
- [ ] `rebuild-and-deploy.yml` workflow
- [ ] `release.yml` workflow
- [ ] Alpha deployment working
- [ ] Staging deployment working
- [ ] Production deployment working
- [ ] Basic documentation

### Nice to Have
- [ ] AI code review integration
- [ ] `promote-to-staging.yml` workflow
- [ ] Git aliases distributed to team
- [ ] Local rebuild scripts
- [ ] Comprehensive documentation
- [ ] Video walkthrough
- [ ] Slack/Discord notifications

### Future Enhancements
- [ ] Automatic promotion to `staging` after N days in `dev`
- [ ] Automatic rollback on failed health checks
- [ ] Feature flags integration
- [ ] Preview environments per PR
- [ ] Performance benchmarking in CI
- [ ] Dependency update automation

---

## Timeline Estimate

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 1: Repository Setup | 1 day | None |
| Phase 2: CI Workflows | 2-3 days | Phase 1 |
| Phase 3: Integration Branch Workflows | 2-3 days | Phase 2 |
| Phase 4: Release Workflow | 1-2 days | Phase 3 |
| Phase 5: Developer Tooling | 1-2 days | Phase 2 |
| Phase 6: Deployment Infrastructure | 2-5 days | Varies |
| Phase 7: Testing & Validation | 2-3 days | All above |
| Phase 8: Team Onboarding | 1-2 weeks | Phase 7 |

**Total: 2-4 weeks** depending on deployment complexity and team size.

---

## Resources

### References
- [Gitworkflow Documentation](https://github.com/rocketraman/gitworkflow)
- [Git Manpage: gitworkflows](https://git-scm.com/docs/gitworkflows)
- [Rocketraman's Git Aliases](https://gist.github.com/rocketraman/1fdc93feb30aa00f6f3a9d7d732102a9)
- [git-reintegrate Tool](https://github.com/felipec/git-reintegrate)

### Tools
- [bumpp](https://github.com/antfu-collective/bumpp) - Version bumping
- [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog) - Changelog generation
- [GitHub CLI (gh)](https://cli.github.com/) - GitHub automation

---

## Notes

_Use this section to track decisions, blockers, and learnings during implementation._

### Decisions
- 

### Blockers
- 

### Learnings
- 
