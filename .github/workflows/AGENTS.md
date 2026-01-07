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

## MAINT BRANCHES

| Branch | Purpose |
|--------|---------|
| `maint` | Current release line, receives hotfixes |
| `maint-X.Y` | Older supported release lines (e.g., `maint-1.2`) |

**Lifecycle:**
- `v1.2.0` release → `maint` moves to that commit
- `v1.3.0` release → old `maint` preserved as `maint-1.2`, new `maint` at v1.3.0
- Patch releases (`v1.2.1`) cut from respective maint branch

**Manual cleanup only** - delete `maint-X.Y` when release line reaches EOL.

## CONVENTIONS

- **Bot token**: Use `BOT_APP_ID`/`BOT_PRIVATE_KEY` for label operations (GITHUB_TOKEN can't trigger workflows)
- **Concurrency**: `rebuild-dev` group, no cancel-in-progress (queue rebuilds)
- **Force push**: `--force-with-lease` only, never bare `--force`
- **rerere**: Enabled in all git configs
- **Build artifacts**: Build once per environment, share via `actions/upload-artifact@v4`
- **Artifact retention**: 1 day (ephemeral, consumed immediately by deploy jobs)

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

## BUILD PIPELINE

```
rebuild-dev → build-dev → deploy-alpha
                ↓
            dev-build artifact (1 day retention)

promote-to-staging/rebuild-staging → build-staging → deploy-staging
                                          ↓
                                  staging-build artifact

release → build-release → deploy-production
              ↓
        release-build artifact
```

## NOTES

- **AI review**: Dormant job in pr-checks (uncomment + add `ANTHROPIC_API_KEY`)
- **Deploy commands**: All `echo` placeholders - implement actual deployment
- **Maint merge**: release.yml auto-merges maint→master before release
- **Maint preservation**: On minor/major release, old maint becomes maint-X.Y
