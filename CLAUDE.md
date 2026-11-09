# Entity Service

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

Shared backend library for multi-tenant entity/organization management.

**Package**: `@sudobility/entity_service` — published to **GitHub Packages** (`npm.pkg.github.com`), not the public npm registry. `.npmrc` scopes `@sudobility` to that registry; installs need a token with `read:packages`.

## Tech Stack

- **Language**: TypeScript (strict mode)
- **Runtime**: Bun
- **Package Manager**: Bun (do not use npm/yarn/pnpm for installing dependencies)
- **Build**: TypeScript compiler (ESM-only)
- **Test**: Vitest
- **Database**: PostgreSQL with Drizzle ORM
- **Framework**: Hono middleware

## Project Structure

```
src/
├── index.ts                        # Main exports
├── types/
│   └── index.ts                    # Re-exports from @sudobility/types + internal config types
├── helpers/
│   ├── index.ts                    # Helper barrel exports
│   ├── EntityHelper.ts             # Entity CRUD operations
│   ├── EntityMemberHelper.ts       # Member management
│   ├── InvitationHelper.ts         # Invitation handling
│   ├── PermissionHelper.ts         # Permission checks
│   ├── ApiKeyHelper.ts             # Entity API key CRUD + verifyKey
│   └── *.test.ts                   # Vitest suites (EntityHelper, ApiKeyHelper)
├── middleware/
│   ├── index.ts                    # Middleware barrel exports
│   └── hono.ts                     # Entity context + require-permission/role + createEntityHelpers
├── schema/
│   └── entities.ts                 # Schema factories (+ *Public variants), default tables, init SQL
├── migrations/
│   ├── index.ts                    # Migration barrel exports
│   └── 001_add_entities.ts         # Entity tables migration + user migration
└── utils/
    ├── index.ts                    # Utils barrel exports
    ├── slug-generator.ts           # Slug generation, validation, invitation tokens
    ├── api-key-generator.ts        # generateApiKey / hashApiKey (SHA-256) / validateKeyPrefix
    └── *.test.ts                   # Vitest suites (slug-generator, api-key-generator)
```

## Commands

```bash
bun run build        # Build ESM-only (tsc tsconfig.esm.json + fix-esm-imports.mjs)
bun run verify       # typecheck -> lint -> build (use before commit; NO tests)
bun run test:unit    # Run tests (Vitest). NOT `bun test` -- see gotchas
bun run typecheck    # tsc --noEmit
bun run lint         # ESLint
bun run clean        # Remove dist/
```

## Key Concepts

### Entities
- **Personal Entity**: Auto-created for each user (entitySlug = random 8-char alphanumeric)
- **Organization**: Created by users, has members with roles

### Roles & Permissions

| Permission | Owner | Manager | Member |
|------------|-------|---------|--------|
| View entity | Yes | Yes | Yes |
| Edit entity | Yes | Yes | No |
| Delete entity | Yes | No | No |
| Manage members | Yes | No | No |
| Invite members | Yes | No | No |
| Manage projects | Yes | Yes | No |
| Create projects | Yes | Yes | No |
| View projects | Yes | Yes | Yes |

### Invitation Flow
1. Owner creates invitation with email + role
2. Invitation stored with entity reference
3. On user signup, pending invitations auto-accepted
4. User becomes member of invited entities

## Public API (entry points)

All exported from the package root (`src/index.ts`). See README for usage examples.

- **`createEntityHelpers(config)`** → `{ entity, members, invitations, permissions, apiKeys? }`.
  `apiKeys` is present only when `config.apiKeysTable` (and `keyPrefix`) are supplied.
- **Schema factories**: `create{Entities,EntityMembers,EntityInvitations,EntityApiKeys}Table(schema, indexPrefix)`
  for a named `pgSchema`, plus `*Public(indexPrefix)` variants and ready-made default tables
  (`entities`, `entityMembers`, ...). `initEntityTables` emits raw `CREATE TABLE` SQL.
- **Middleware** (`./middleware/hono`): `createEntityContextMiddleware`,
  `createRequirePermissionMiddleware`, `createRequireRoleMiddleware`.
- **Utils**: slug/token generation + API-key generation/hashing.
- **Types & permission constants** re-exported from `@sudobility/types`.

Peer deps (installed by the consumer, never in `dependencies`): `drizzle-orm`, `hono`, `@sudobility/types`.

## Publishing

Publishing is normally automatic: the `ci-cd.yml` workflow (reusable
`johnqh/workflows` with `publish-target: github`) publishes to GitHub Packages
on push to `main`/`develop`. Bump `version` in package.json first. A manual
publish still targets GitHub Packages because of `.npmrc`:

```bash
bun run verify       # typecheck + lint + build
npm publish          # -> npm.pkg.github.com (NOT public npm)
```

## Architecture

```
entity_service (this package)
    ↑
shapeshyft_api (backend)
sudojo_api (backend)
```

## Code Patterns

- Error conventions: invalid/unknown entity slug → 404; permission denied → 403; duplicate invitation → 409.
- Types import as `type`; enums/helpers (`EntityRole`, `hasPermission`) import as values.

## Workspace Context

This project is part of the **ShapeShyft** multi-project workspace. Its direct downstream consumer is `shapeshyft_api` (see below).

## Downstream Impact

| Downstream Consumer | Relationship |
|---------------------|-------------|
| `shapeshyft_api` | Direct dependency - uses entity middleware, helpers, and schema factories |

After making changes:
1. `bun run verify` in this project
2. Bump `version` in package.json and push to `main` (CI publishes to GitHub Packages)
3. In `shapeshyft_api`: `bun update @sudobility/entity_service` then rebuild

Note: `entity_pages` does NOT depend on this package. It uses `@sudobility/entity_client` (a separate repo).

## Local Dev Workflow

```bash
# In this project:
bun link

# In shapeshyft_api:
bun link @sudobility/entity_service

# Rebuild after changes:
bun run build

# When done, unlink:
bun unlink @sudobility/entity_service && bun install
```

## Pre-Commit Checklist

```bash
bun run verify     # Runs: typecheck -> lint -> build (does NOT include tests)
bun run test:unit  # Run tests separately (Vitest)
```

## Gotchas

- **Schema factory functions require a schema name argument** -- `createEntitiesTable(mySchema, 'my_app')`. Forgetting the prefix causes table name collisions.
- **Peer dependencies (drizzle-orm, hono) must be installed by consumers** -- do not add to `dependencies`.
- **Personal entities use randomly generated slugs** (8-char alphanumeric via `generateEntitySlug()`). Slugs are not derived from userId.
- **Role permissions are defined in `@sudobility/types`** and re-exported via `types/index.ts`. Changing them affects all consumers.
- **Tests run on Vitest, not Bun's runner.** Use `bun run test:unit`; `bun test` loads Bun's native runner which does not execute the `vitest`-based suites correctly.
- **Build needs the ESM-import fixup.** `scripts/fix-esm-imports.mjs` rewrites extensionless/relative specifiers to `.js` after `tsc`; `tsc` alone produces output Node's ESM loader rejects. `tsconfig.cjs.json` exists but is not used by any build script (ESM-only output).
- **GitHub Packages, not npm.** Installing/publishing requires the `@sudobility` scope pointed at `npm.pkg.github.com` (via `.npmrc`) and a token with `read:packages` (consumers) / `write:packages` (publish).

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
