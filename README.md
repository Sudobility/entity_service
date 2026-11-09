# @sudobility/entity_service

Shared backend library for multi-tenant entity/organization management with Drizzle ORM and Hono middleware.

## Installation

Published to **GitHub Packages** (not the public npm registry). Consumers must
point the `@sudobility` scope at the GitHub registry — this repo's `.npmrc` does:

```
@sudobility:registry=https://npm.pkg.github.com
```

Install with an authenticated token (a PAT with `read:packages`):

```bash
bun add @sudobility/entity_service
```

`drizzle-orm` and `hono` are peer dependencies and must be installed by the
consuming app.

## Usage

```typescript
import {
  createEntityHelpers,
  createEntitiesTable,
  createEntityMembersTable,
  createEntityContextMiddleware,
} from '@sudobility/entity_service';

// Setup helpers
const helpers = createEntityHelpers({
  db: drizzleDb,
  entitiesTable: schema.entities,
  membersTable: schema.entityMembers,
  invitationsTable: schema.entityInvitations,
  usersTable: schema.users,
});

// Entity operations
const entity = await helpers.entity.getOrCreatePersonalEntity(userId, email);
const org = await helpers.entity.createOrganizationEntity(userId, { displayName: 'My Org' });
const entities = await helpers.entity.getUserEntities(userId);

// Hono middleware
app.use('/api/v1/entities/:entitySlug/*', createEntityContextMiddleware(config, {
  getUserId: (c) => c.get('userId'),
}));
```

## API

### Schema Factories

| Export | Description |
|--------|-------------|
| `createEntitiesTable(pgSchema, prefix)` | Entities table definition |
| `createEntityMembersTable(pgSchema, prefix)` | Entity members table |
| `createEntityInvitationsTable(pgSchema, prefix)` | Invitations table |
| `createEntityApiKeysTable(pgSchema, prefix)` | Entity API keys table |

### Helpers

| Helper | Key Methods |
|--------|-------------|
| `EntityHelper` | `getOrCreatePersonalEntity`, `createOrganizationEntity`, `getUserEntities`, CRUD |
| `EntityMemberHelper` | Member listing, role updates, removal |
| `InvitationHelper` | Create, accept, decline, cancel, renew invitations |
| `PermissionHelper` | Role-based permission checks (Owner > Manager > Member) |
| `ApiKeyHelper` | `createKey`, `getKeys`, `updateKey`, `revokeKey`, `verifyKey` |

Entity API keys authenticate a caller as the entity itself (CI, scripts,
integrations) rather than as a member. Only the SHA-256 hash is stored, so the
plaintext is returned once by `createKey` and cannot be recovered -- a lost key
is rotated, not revealed. Gate writes on `canManageApiKeys` and reads on
`canViewApiKeys`; the helper does not check permissions itself.

```typescript
const helpers = createEntityHelpers({ ...config, apiKeysTable: schema.entityApiKeys, keyPrefix: 'shyft' });
const { key } = await helpers.apiKeys!.createKey(entityId, userId, 'CI deploy');
const identity = await helpers.apiKeys!.verifyKey(incomingKey);
```

### Middleware

| Export | Description |
|--------|-------------|
| `createEntityContextMiddleware` | Hono middleware injecting entity context, role, and permissions |
| `createRequirePermissionMiddleware` | Gate a route on a specific permission (403 if missing) |
| `createRequireRoleMiddleware` | Gate a route on a minimum role (403 if below) |
| `createEntityHelpers` | Factory returning `{ entity, members, invitations, permissions, apiKeys? }` |

### Types

Re-exports from `@sudobility/types`: `Entity`, `EntityMember`, `EntityInvitation`, `EntityType`, `EntityRole`, `EntityPermissions`

## Development

```bash
bun run build        # Build ESM-only (tsc + scripts/fix-esm-imports.mjs)
bun run verify       # typecheck + lint + build (use before commit; no tests)
bun run test:unit    # Run tests (Vitest) -- NOT `bun test`
bun run test:watch   # Vitest in watch mode
bun run typecheck    # TypeScript check (tsc --noEmit)
bun run lint         # ESLint
bun run format       # Prettier write
bun run clean        # Remove dist/
```

> Use `bun run test:unit`, not `bun test`: the suites import from `vitest`, so
> Bun's built-in test runner will not run them correctly.

## License

BUSL-1.1
