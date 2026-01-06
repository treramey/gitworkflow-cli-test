# AGENTS.md

Instructions for AI agents working in this repository.

## Project Overview

Turborepo monorepo using Better-T-Stack with React, TanStack Start (SSR), and TypeScript.

**Structure:**
- `apps/web/` - React frontend (TanStack Start + Vite)
- `packages/config/` - Shared TypeScript config
- `packages/env/` - Type-safe environment variables (t3-oss/env-core + Zod)

## Build/Dev/Test Commands

```bash
# Install dependencies
bun install

# Development
bun run dev              # All apps
bun run dev:web          # Web app only (port 3001)

# Build & Type Check
bun run build            # Build all
bun run check-types      # TypeScript check all

# Single package commands (use turbo filter)
turbo -F web build       # Build web only
turbo -F web check-types # Type check web only

# Run commands in specific workspace
bun run --cwd apps/web build
```

### Testing

No test runner configured yet. Testing libraries installed in web app:
- `@testing-library/react`
- `@testing-library/dom`
- `jsdom`

When adding tests, use Vitest:
```bash
# Single test file (once vitest is configured)
bun run --cwd apps/web vitest run path/to/test.test.ts
bun run --cwd apps/web vitest run --testNamePattern="test name"
```

## Code Style

### TypeScript

**Strict mode enabled.** Key compiler options:
- `strict: true`
- `noUncheckedIndexedAccess: true` - array/object access may be undefined
- `noUnusedLocals: true`
- `noUnusedParameters: true`
- `verbatimModuleSyntax: true` - use `import type` for type-only imports

**Type Safety Rules:**
- No `any` - use `unknown` and narrow
- No non-null assertion (`!`) - handle undefined explicitly
- No type assertions (`as Type`) - use type guards or discriminated unions
- Model domain with discriminated unions; parse inputs at boundaries

### Imports

Order imports (group with blank lines):
1. External packages (react, libraries)
2. Internal aliases (`@/...`)
3. Relative imports (`./`, `../`)

```typescript
// External
import { useState } from "react";
import { z } from "zod";

// Internal alias
import { cn } from "@/lib/utils";
import { Button } from "@/components/ui/button";

// Relative
import { localHelper } from "./helpers";
```

Use `import type` for type-only imports:
```typescript
import type { VariantProps } from "class-variance-authority";
```

Path alias: `@/*` maps to `./src/*` in web app.

### Naming Conventions

- **Files:** kebab-case (`button.tsx`, `dropdown-menu.tsx`)
- **Components:** PascalCase functions, default export for pages/routes
- **Functions:** camelCase
- **Constants:** UPPER_SNAKE_CASE for true constants
- **Types/Interfaces:** PascalCase

### React Components

Function declarations preferred. Props spread at end:
```typescript
function Button({
  className,
  variant = "default",
  size = "default",
  ...props
}: ButtonPrimitive.Props & VariantProps<typeof buttonVariants>) {
  return (
    <ButtonPrimitive
      data-slot="button"
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  );
}
```

Use `data-slot` for component identification in UI primitives.

### Styling

**Tailwind CSS v4** with shadcn/ui (base-lyra style).

- Use `cn()` from `@/lib/utils` for conditional classes
- CSS variables defined in `apps/web/src/index.css`
- Dark mode: class-based (`.dark`)
- Icon library: Lucide React

```typescript
import { cn } from "@/lib/utils";

<div className={cn("base-classes", condition && "conditional-class", className)} />
```

### Environment Variables

Use `@gitworkflow-ci-test/env` package with Zod schemas:
- Server: `packages/env/src/server.ts`
- Client: `packages/env/src/web.ts` (must prefix with `VITE_`)

```typescript
import { env } from "@gitworkflow-ci-test/env/server";
```

### Error Handling

- Parse and validate at boundaries (API responses, user input)
- Use Zod for runtime validation
- Discriminated unions for error states
- No throwing for expected failures; return Result types when appropriate

### File Organization (Web App)

```
apps/web/src/
├── components/
│   ├── ui/          # shadcn/ui primitives
│   └── *.tsx        # App components
├── lib/
│   └── utils.ts     # Utilities (cn, etc.)
├── routes/
│   ├── __root.tsx   # Root layout
│   └── index.tsx    # Home page
├── index.css        # Global styles + CSS variables
└── router.tsx       # TanStack Router config
```

### TanStack Router

File-based routing in `apps/web/src/routes/`:
```typescript
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/")({
  component: HomeComponent,
});

function HomeComponent() {
  return <div>...</div>;
}
```

## Git & SCM

- Check for `.jj/` before VCS commands; use jj if present
- Concise commit messages; sacrifice grammar for brevity
- Never add AI to attribution in commits/PRs
- `gh` CLI available for GitHub operations

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `@tanstack/react-router` | File-based routing |
| `@tanstack/react-start` | SSR framework |
| `@tanstack/react-query` | Server state |
| `@tanstack/react-form` | Form handling |
| `@base-ui/react` | Headless UI primitives |
| `class-variance-authority` | Variant styles |
| `zod` | Schema validation |
| `lucide-react` | Icons |

## Common Patterns

### Variant Components (CVA)
```typescript
const variants = cva("base-classes", {
  variants: {
    variant: { default: "...", outline: "..." },
    size: { default: "...", sm: "..." },
  },
  defaultVariants: { variant: "default", size: "default" },
});
```

### Shadcn UI Components

Add via CLI:
```bash
cd apps/web && bunx shadcn@latest add [component]
```

Components use `@base-ui/react` primitives, not Radix.
