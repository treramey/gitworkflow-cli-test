# WORKFLOWS KNOWLEDGE BASE

## OVERVIEW

Gitworkflow CI/CD: ephemeral integration branches rebuilt by CI, label-driven promotion.

## WHERE TO LOOK

| Task | File | Notes |
|------|------|-------|
| PR validation | `pr-checks.yml` | Build + labels + merge gate |
| Dev/staging rebuild | `rebuild-and-deploy.yml` | Triggered by labels, schedule, dispatch |
| Manual promotion | `promote-to-staging.yml` | Release manager promotes topic |
| Release automation | `release.yml` | On PR merge to master |

## WORKFLOW TRIGGERS

| Workflow | Triggers |
|----------|----------|
| pr-checks | PR to master (opened/sync/review) |
| rebuild-and-deploy | `env: dev` label, schedule (2h), dispatch, PR closed |
| promote-to-staging | Manual dispatch only |
| release | PR merged to master |

## LABEL SEMANTICS

| Label | Meaning | Set By |
|-------|---------|--------|
| `status: ci-passed` | Build succeeded | pr-checks |
| `status: ci-failed` | Build failed | pr-checks |
| `status: needs-review` | Awaiting approval | pr-checks |
| `status: conflict` | Merge conflict in rebuild | rebuild-and-deploy |
| `env: dev` | Ready for alpha | pr-checks (CI + approval) |
| `env: staging` | Promoted to staging | promote-to-staging |

## CONVENTIONS

- **Bot token**: Use `BOT_APP_ID`/`BOT_PRIVATE_KEY` for label operations (GITHUB_TOKEN can't trigger workflows)
- **Concurrency**: `rebuild-dev` group, no cancel-in-progress (queue rebuilds)
- **Force push**: `--force-with-lease` only, never bare `--force`
- **rerere**: Enabled in all git configs

## ANTI-PATTERNS

- **DO NOT** add deployment secrets without updating deploy steps
- **DO NOT** change label names without updating all workflows
- **DO NOT** skip `should-run` job gating

## MERGE GATE

`require-staging-approval` job blocks merge until:
1. `status: ci-passed`
2. Code review approval
3. `env: staging` label

## CONFLICT HANDLING

On merge conflict during rebuild:
1. Abort merge for that topic
2. Remove `env: dev`, add `status: conflict`
3. Post fix instructions to PR
4. Continue with other topics

## NOTES

- **AI review**: Dormant job in pr-checks (uncomment + add `ANTHROPIC_API_KEY`)
- **Deploy commands**: All `echo` placeholders - implement actual deployment
- **Maint merge**: release.yml auto-merges maint→master before release
