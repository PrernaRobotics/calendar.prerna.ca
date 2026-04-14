# Prerna Calendar — calendar.prerna.ca

Self-hosted cal.com instance for **Prerna Robotics Inc.** providing external booking/scheduling and internal team calendar at `calendar.prerna.ca`.

## Project Context

- **Company**: Prerna Robotics Inc. (prerna.ca)
- **Subdomain**: calendar.prerna.ca
- **Source**: Forked from [cal.com/cal.com](https://github.com/calcom/cal.com) (cloned 2026-04-12)
- **Goal**: Full self-hosted cal.com with Prerna branding — external client booking (demo scheduling for PREP product) + internal team scheduling
- **Parent site**: prerna.ca (React SPA on AWS Amplify, ca-central-1)

## Infrastructure

- **Hosting target**: AWS Amplify (separate app for the subdomain) with external RDS PostgreSQL
- **DNS**: Route53 hosted zone `Z0039219V8R9ZTDCN4Y2` (same zone as prerna.ca)
- **Parent Amplify App ID**: `d10r3ez5qh4pl6` (for reference — this is the prerna.ca app, not this one)
- **Region**: ca-central-1 (same as parent site)
- **GitHub repo**: `PrernaRobotics/calendar.prerna.ca` (private)

### AWS Resources (Provisioned)

| Resource | Identifier | Endpoint |
|----------|-----------|----------|
| RDS PostgreSQL 15 | `prerna-calendar-db` (db.t4g.micro, 20GB gp3) | `prerna-calendar-db.cxcwu0cocliu.ca-central-1.rds.amazonaws.com:5432` |
| ElastiCache Redis (serverless) | `prerna-calendar-redis` | `prerna-calendar-redis-tks45w.serverless.cac1.cache.amazonaws.com:6379` |
| Security Group (RDS) | `sg-053baaf7c17b81cad` (prerna-calendar-rds) | Inbound TCP 5432 from 0.0.0.0/0 |
| Security Group (Redis) | `sg-0bed2bf62eb96fd6c` (prerna-calendar-redis) | Inbound TCP 6379 from 0.0.0.0/0 |

- Production env values are in `.env.production` (gitignored)

## Local Development

### Prerequisites

- Docker Desktop (required)
- Node.js >= 18
- Yarn (bundled via `.yarn/releases/yarn-4.12.0.cjs`)

### Quick Start (Native Dev — recommended)

Port 3000 is used by another local service (prep-web), so cal.com runs on **port 3001**.

```bash
cd ~/Coding/calendar.prerna.ca/Prerna\ Calendar

# 1. Start PostgreSQL + Redis via Docker
docker-compose up -d database redis

# 2. Install dependencies (~1-2 min)
yarn install

# 3. Run migrations + generate types
npx prisma migrate deploy --schema packages/prisma/schema.prisma
yarn prisma generate

# 4. Start dev server (http://localhost:3001)
yarn dev
```

### Quick Start (Full Docker)

```bash
cd ~/Coding/calendar.prerna.ca/Prerna\ Calendar
docker-compose up
```

This starts PostgreSQL (port 5450), Redis, and the cal.com web app (port 3001). First build takes ~5-10 minutes.

### Environment Variables (Key Ones)

Already configured in `.env`:

- `DATABASE_URL` — `postgresql://postgres:postgres@localhost:5450/calendso`
- `DATABASE_DIRECT_URL` — Same as DATABASE_URL (no connection pooler locally)
- `NEXTAUTH_SECRET` — Generated, do not commit
- `CALENDSO_ENCRYPTION_KEY` — 32-char AES256 key, do not commit
- `NEXT_PUBLIC_WEBAPP_URL` — `http://localhost:3001` (local) or `https://calendar.prerna.ca` (prod)
- `PORT=3001` — Dev server port (3000 is taken by prep-web)
- `CALCOM_TELEMETRY_DISABLED=1` — Telemetry off

### Docker Compose Services

| Service | Host Port | Description |
|---------|-----------|-------------|
| database | 5450 | PostgreSQL (postgres:15-alpine) |
| redis | 6379 | Redis cache |
| calcom | 3001 | Main Next.js app |
| calcom-api | 80 | API v2 |
| studio | 5555 | Prisma Studio (dev only) |

### Notes

- `packages/prisma/.env` must NOT be a symlink to root `.env` (causes Prisma conflict). It should be empty or contain only `DATABASE_URL`.
- The `.env.example` dotenv-checker auto-adds missing keys to `.env` on `yarn dev` — this is normal.

## What's Been Done

1. Cloned cal.com repo into `~/Coding/calendar.prerna.ca/Prerna Calendar/`
2. Created `.env` with generated secrets (`NEXTAUTH_SECRET`, `CALENDSO_ENCRYPTION_KEY`), database config, and local dev URLs
3. Docker Compose config fixed — credentials aligned, postgres:15-alpine image, port 5450 exposed
4. Dependencies installed, 590 Prisma migrations applied, types generated
5. Dev server runs on port 3001 (`http://localhost:3001`)

## What's Next

1. **Branding**: Customize logo, colors, app name for Prerna Robotics
2. **AWS Setup**: Create RDS PostgreSQL, new Amplify app, Route53 subdomain record for `calendar.prerna.ca`
3. **Deploy**: Push to Amplify with production `.env` pointing to RDS
4. **Customize**: Add Prerna-specific event types, booking pages, team setup

## Branding Changes (Applied)

- App name: "Prerna Calendar" (env vars + constants)
- Company name: "Prerna Robotics Inc."
- Colors: Extracted from prerna.ca CSS variables:
  - Primary brand: `#2563eb` (prep-blue)
  - Accent: `#7c3aed` (prep-violet)
  - Surfaces: `#09090b` / `#18181b` / `#1c1c1f`
  - Applied to `--cal-brand` in light mode (`hsla(217,91%,53%,1)`) and dark mode (`hsla(217,91%,60%,1)`)
- Default timezone: America/Toronto (EST)
- Allowed hostnames: `calendar.prerna.ca`, `localhost:3001`
- Logo: TODO — replace Cal.com logos in `apps/web/public/` with Prerna Robotics assets

## Important Notes

- `.env` is gitignored — never commit secrets
- The upstream cal.com CLAUDE.md dev guide is preserved below for engineering standards
- Cal.com is AGPL-licensed — the codebase must stay open source if modified
- Enterprise features (SAML, org management) require a license key from cal.com/sales

---

# Cal.com Development Guide for AI Agents

You are a senior Cal.com engineer working in a Yarn/Turbo monorepo. You prioritize type safety, security, and small, reviewable diffs.

## Do

- Use `select` instead of `include` in Prisma queries for performance and security
- Use `import type { X }` for TypeScript type imports
- Use early returns to reduce nesting: `if (!booking) return null;`
- Use `ErrorWithCode` for errors in non-tRPC files (services, repositories, utilities); use `TRPCError` only in tRPC routers
- Use conventional commits: `feat:`, `fix:`, `refactor:`
- Create PRs in draft mode by default
- Run `yarn type-check:ci --force` before concluding CI failures are unrelated to your changes
- Import directly from source files, not barrel files (e.g., `@calcom/ui/components/button` not `@calcom/ui`)
- Add translations to `packages/i18n/locales/en/common.json` for all UI strings
- Use `date-fns` or native `Date` instead of Day.js when timezone awareness isn't needed
- Put permission checks in `page.tsx`, never in `layout.tsx`
- Use `ast-grep` for searching if available; otherwise use `rg` (ripgrep), then fall back to `grep`
- Use Biome for formatting and linting
- Only add code comments that explain **why**, not **what** — see [code comment guidelines](agents/rules/quality-code-comments.md)


## Don't

- Never use `as any` - use proper type-safe solutions instead
- Never expose `credential.key` field in API responses or queries
- Never commit secrets or API keys
- Never modify `*.generated.ts` files directly - they're created by app-store-cli
- Never put business logic in repositories - that belongs in Services
- Never use barrel imports from index.ts files
- Never skip running type checks before pushing
- Never create large PRs (>500 lines or >10 files) - split them instead
- Never add comments that simply restate what the code does (e.g., `// Get the user` above a `getUser()` call)

## PR Size Guidelines

Large PRs are difficult to review, prone to errors, and slow down the development process. Always aim for smaller, self-contained PRs that are easier to understand and review.

### Size Limits

- **Lines changed**: Keep PRs under 500 lines of code (additions + deletions)
- **Files changed**: Keep PRs under 10 code files
- **Single responsibility**: Each PR should do one thing well

**Note**: These limits apply to code files only. Non-code files like documentation (README.md, CHANGELOG.md), lock files (yarn.lock, package-lock.json), and auto-generated files are excluded from the count.

### How to Split Large Changes

When a task requires extensive changes, break it into multiple PRs:

1. **By layer**: Separate database/schema changes, backend logic, and frontend UI into different PRs
2. **By feature component**: Split a feature into its constituent parts (e.g., API endpoint PR, then UI PR, then integration PR)
3. **By refactor vs feature**: Do preparatory refactoring in a separate PR before adding new functionality
4. **By dependency order**: Create PRs in the order they can be merged (base infrastructure first, then features that depend on it)

### Examples of Good PR Splits

**Instead of one large "Add booking notifications" PR:**
- PR 1: Add notification preferences schema and migration
- PR 2: Add notification service and API endpoints
- PR 3: Add notification UI components
- PR 4: Integrate notifications into booking flow

**Instead of one large "Refactor calendar sync" PR:**
- PR 1: Extract calendar sync logic into dedicated service
- PR 2: Add new calendar provider abstraction
- PR 3: Migrate existing providers to new abstraction
- PR 4: Add new calendar provider support

### Benefits of Smaller PRs

- Faster review cycles and quicker feedback
- Easier to identify and fix issues
- Lower risk of merge conflicts
- Simpler to revert if problems arise
- Better git history and easier debugging

## Commands

See [agents/commands.md](agents/commands.md) for full reference. Key commands:

```bash
yarn type-check:ci --force  # Type check (always run before pushing)
yarn biome check --write .  # Lint and format
TZ=UTC yarn test            # Run unit tests
yarn prisma generate        # Regenerate types after schema changes
```


## Boundaries

### Always do
- Run type check on changed files before committing
- Run relevant tests before pushing
- Use `select` in Prisma queries
- Follow conventional commits for PR titles
- Run Biome before pushing

### Ask first
- Adding new dependencies
- Schema changes to `packages/prisma/schema.prisma`
- Changes affecting multiple packages
- Deleting files
- Running full build or E2E suites

### Never do
- Commit secrets, API keys, or `.env` files
- Expose `credential.key` in any query
- Use `as any` type casting
- Force push or rebase shared branches
- Modify generated files directly

## Project Structure

```
apps/web/                    # Main Next.js application
packages/prisma/             # Database schema (schema.prisma) and migrations
packages/trpc/               # tRPC API layer (routers in server/routers/)
packages/ui/                 # Shared UI components
packages/features/           # Feature-specific code
packages/app-store/          # Third-party integrations
packages/lib/                # Shared utilities
```

### Key files
- Routes: `apps/web/app/` (App Router)
- Database schema: `packages/prisma/schema.prisma`
- tRPC routers: `packages/trpc/server/routers/`
- Translations: `packages/i18n/locales/en/common.json`
- Workflow constants: `packages/features/ee/workflows/lib/constants.ts`

## Tech Stack

- **Framework**: Next.js 13+ (App Router in some areas)
- **Language**: TypeScript (strict)
- **Database**: PostgreSQL with Prisma ORM
- **API**: tRPC for type-safe APIs
- **Auth**: NextAuth.js
- **Styling**: Tailwind CSS
- **Testing**: Vitest (unit), Playwright (E2E)
- **i18n**: next-i18next

## Code Examples

### Good error handling

```typescript
// Good - Descriptive error with context
throw new Error(`Unable to create booking: User ${userId} has no available time slots for ${date}`);

// Bad - Generic error
throw new Error("Booking failed");
```

For which error class to use (`ErrorWithCode` vs `TRPCError`) and concrete examples, see [quality-error-handling](agents/rules/quality-error-handling.md).

### Good Prisma query

```typescript
// Good - Use select for performance and security
const booking = await prisma.booking.findFirst({
  select: {
    id: true,
    title: true,
    user: {
      select: {
        id: true,
        name: true,
        email: true,
      }
    }
  }
});

// Bad - Include fetches all fields including sensitive ones
const booking = await prisma.booking.findFirst({
  include: { user: true }
});
```

### Good imports

```typescript
// Good - Type imports and direct paths
import type { User } from "@prisma/client";
import { Button } from "@calcom/ui/components/button";

// Bad - Regular import for types, barrel imports
import { User } from "@prisma/client";
import { Button } from "@calcom/ui";
```

### API v2 Imports (apps/api/v2)

When importing from `@calcom/features` or `@calcom/trpc` into `apps/api/v2`, **do not import directly** because the API v2 app's `tsconfig.json` doesn't have path mappings for these modules, which causes "module not found" errors.

Instead, re-export from `packages/platform/libraries/index.ts` and import from `@calcom/platform-libraries`:

```typescript
// Step 1: In packages/platform/libraries/index.ts, add the export
export { ProfileRepository } from "@calcom/features/profile/repositories/ProfileRepository";

// Step 2: In apps/api/v2, import from platform-libraries
import { ProfileRepository } from "@calcom/platform-libraries";

// Bad - Direct import causes module not found error in apps/api/v2
import { ProfileRepository } from "@calcom/features/profile/repositories/ProfileRepository";
```

## PR Checklist

- [ ] Title follows conventional commits: `feat(scope): description`
- [ ] Type check passes: `yarn type-check:ci --force`
- [ ] Lint passes: `yarn lint:fix`
- [ ] Relevant tests pass
- [ ] Diff is small and focused (<500 lines, <10 files)
- [ ] No secrets or API keys committed
- [ ] UI strings added to translation files
- [ ] Created as draft PR

## When Stuck

- Ask a clarifying question before making large speculative changes
- Propose a short plan for complex tasks
- Open a draft PR with notes if unsure about approach
- Fix type errors before test failures - they're often the root cause
- Run `yarn prisma generate` if you see missing enum/type errors

## Spec-Driven Development (Opt-In)

For complex features, you can use spec-driven development when explicitly requested.

**To enable:** Tell the AI "use spec-driven development" or "follow the spec workflow"

See [SPEC-WORKFLOW.md](SPEC-WORKFLOW.md) for the full workflow documentation.

## Extended Documentation

For detailed information, see the `agents/` directory:

- **[agents/README.md](agents/README.md)** - Rules index and architecture overview
- **[agents/rules/](agents/rules/)** - Modular engineering rules
- **[agents/commands.md](agents/commands.md)** - Complete command reference
- **[agents/knowledge-base.md](agents/knowledge-base.md)** - Domain knowledge and business rules
