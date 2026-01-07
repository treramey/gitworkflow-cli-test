# PROJECT KNOWLEDGE BASE

**Generated:** 2026-01-07
**Commit:** bd5cd6a
**Branch:** tr/test-comments

## OVERVIEW

Gitworkflow CI/CD test bed - TypeScript monorepo (Turborepo + Bun) with TanStack Start frontend. Primary purpose: demonstrate ephemeral integration branch workflow.

## STRUCTURE

```
./
├── .github/workflows/   # Gitworkflow CI - see AGENTS.md there
├── apps/web/            # TanStack Start + shadcn/ui frontend
├── packages/
│   ├── config/          # Shared tsconfig.base.json
│   └── env/             # t3-env typed environment (server/web exports)
├── .gitconfig           # Gitworkflow aliases (include in ~/.gitconfig)
└── gitworkflow-architecture.md  # Canonical workflow docs
```

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Add UI component | `apps/web/src/components/ui/` | shadcn/ui pattern |
| Add route | `apps/web/src/routes/` | TanStack Router file-based |
| Add env var | `packages/env/src/server.ts` or `web.ts` | t3-env schema |
| Understand CI | `.github/workflows/` + `gitworkflow-architecture.md` | |
| Git aliases | `.gitconfig` | `git where`, `git topics`, `git rebuild-dev` |

## CONVENTIONS

### Git Workflow (CRITICAL)
- **Branch from master ONLY** - never from staging/dev
- **Never commit to dev/staging** - they are ephemeral, rebuilt by CI
- **Topic branch format**: `<initials>/<feature-name>` (e.g., `jd/add-auth`)
- **Single PR to master** - labels control promotion through environments
- **Conventional commits**: `feat:`, `fix:`, `docs:`, etc.

### Integration Branches
| Branch | Purpose | Rebuilt |
|--------|---------|---------|
| `master` | Production, permanent | Never |
| `maint` | Current hotfix line | After releases |
| `maint-X.Y` | Older supported releases (e.g., `maint-1.2`) | Manual patches |
| `staging` | Beta/QA | On promotion |
| `dev` | Alpha/proposed | Every 2h + on label |

### Topic Status (`git where`)
- `.` not integrated
- `-` in dev (alpha)
- `+` in staging (beta)
- `*` in maint
- `=` graduated (master)

### TypeScript
- **Strict mode** + `noUncheckedIndexedAccess`
- Workspace packages use `workspace:*` protocol
- Dependency versions via `catalog:` in root package.json

## ANTI-PATTERNS

- **DO NOT** branch from staging/dev
- **DO NOT** commit directly to dev/staging
- **DO NOT** use git hooks that block (`--no-verify` etc)
- **DO NOT** force push to master
- **AVOID** `any`, non-null assertions (`!`), type assertions (`as`)

## UNIQUE STYLES

- **Bun with pnpm catalog**: Uses `catalog:` protocol despite Bun runtime
- **Raw TS exports**: `packages/env` exports `.ts` files directly (consumer transpiles)
- **rerere enabled**: Git remembers conflict resolutions (`.gitconfig`)
- **No tests yet**: Testing deps present but no test files/config

## COMMANDS

```bash
bun install              # Install deps
bun run dev              # Dev server (all apps)
bun run dev:web          # Dev server (web only)
bun run build            # Build all
bun run check-types      # Type check all

# Git workflow
git where                # Show topic status
git topics               # List all topics
git ready-to-graduate    # Topics in staging not master
git rebuild-dev          # Local dev rebuild (test)
```

## LABEL FLOW

```
PR opened → CI runs → status: ci-passed → approval → env: dev
                                                        ↓
                                              (auto: dev rebuild)
                                                        ↓
                                              manual: env: staging
                                                        ↓
                                              (staging deploy)
                                                        ↓
                                              merge to master → release
```

## NOTES

- **GitHub App token**: Workflows use `BOT_APP_ID`/`BOT_PRIVATE_KEY` for label events
- **AI review**: Commented out in pr-checks.yml (needs `ANTHROPIC_API_KEY`)
- **Deploy placeholders**: Actual deploy commands not implemented
- **apps/web tsconfig**: Doesn't extend shared config (should fix)
