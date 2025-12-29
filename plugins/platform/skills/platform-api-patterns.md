---
name: platform-api-patterns
description: Apply WORKWAY platform API patterns for Hono routes, authentication middleware, domain-based teams, and session management. Use when building API endpoints or modifying platform authentication.
---

# Platform API Patterns

Patterns for building API endpoints in the WORKWAY platform (apps/api).

## Core Stack

| Component | Technology | Location |
|-----------|-----------|----------|
| API Framework | Hono | apps/api/src/index.ts |
| Database | D1 (SQLite) | marketplace-production |
| Session Storage | KV (Workers KV) | SESSIONS namespace |
| Cache | KV (Workers KV) | CACHE namespace |
| Deployment | Cloudflare Workers | api.workway.co |

## Route Structure

### Creating a New Route

1. Create route file in `apps/api/src/routes/`
2. Mount in `apps/api/src/index.ts`
3. Add middleware as needed

**Example: New Feature Route**

```typescript
// File: apps/api/src/routes/feature.ts
import { Hono } from 'hono';
import type { Env } from '../types';
import { requireAuth } from '../lib/sessions';
import { db } from '../lib/db';

const app = new Hono<{ Bindings: Env }>();

// List items (authenticated)
app.get('/', requireAuth, async (c) => {
  const session = c.get('session');

  const items = await db.select()
    .from(schema.items)
    .where(eq(schema.items.userId, session.userId));

  return c.json({ items });
});

// Create item (authenticated)
app.post('/', requireAuth, async (c) => {
  const session = c.get('session');
  const body = await c.req.json();

  const newItem = await db.insert(schema.items).values({
    userId: session.userId,
    ...body,
  }).returning();

  return c.json({ item: newItem }, 201);
});

export default app;
```

```typescript
// File: apps/api/src/index.ts
import feature from './routes/feature';

// Mount route
app.route('/feature', feature);
```

## Authentication Patterns

### Session-Based Auth (Primary)

Sessions stored in Workers KV with automatic expiration.

```typescript
import { requireAuth, optionalAuth, requireRole } from '../lib/sessions';

// Require authentication
app.get('/protected', requireAuth, async (c) => {
  const session = c.get('session');  // Injected by requireAuth
  // session.userId, session.email, session.role available
});

// Optional authentication
app.get('/public-or-private', optionalAuth, async (c) => {
  const session = c.get('session');  // null if not authenticated

  if (session) {
    // Personalized response
  } else {
    // Public response
  }
});

// Require specific role
app.get('/admin-only', async (c) => {
  const session = await requireRole(c, 'ADMIN');
  // Only ADMIN role can access
});
```

### Session Middleware Details

| Middleware | Behavior | Use Case |
|------------|----------|----------|
| `requireAuth` | Throws 401 if no session | Protected endpoints |
| `optionalAuth` | Sets session if present, continues if not | Personalization |
| `requireRole(...roles)` | Throws 403 if role mismatch | Role-based access |

**Session object structure:**

```typescript
interface Session {
  userId: string;
  email: string;
  role: string;          // USER, DEVELOPER, ADMIN
  createdAt: number;
  expiresAt: number;
  lastActive: number;
}
```

## Domain-Based Team Auto-Join

Users automatically join teams based on email domain. This is the core team mechanism.

### How It Works

```typescript
// On signup/login (apps/api/src/routes/auth.ts)
const domain = extractDomain(email);  // user@halfdozen.co → "halfdozen.co"

// Check if team exists
const existingTeam = await db.select()
  .from(teams)
  .where(eq(teams.domain, domain))
  .limit(1);

if (existingTeam.length > 0) {
  // Team exists → add as member
  await db.insert(teamMembers).values({
    teamId: existingTeam[0].id,
    userId: user.id,
    role: 'member',
  });
} else {
  // First user from domain → create team, user becomes owner
  const newTeam = await db.insert(teams).values({
    domain,
    name: domainToTeamName(domain),  // "halfdozen.co" → "Half Dozen"
    ownerId: user.id,
  }).returning();

  await db.insert(teamMembers).values({
    teamId: newTeam[0].id,
    userId: user.id,
    role: 'owner',
  });
}
```

### Team Role Hierarchy

```typescript
const roleHierarchy = { member: 0, admin: 1, owner: 2 };

// Check team membership with minimum role
async function requireTeamMember(
  db: DrizzleClient,
  teamId: string,
  userId: string,
  minRole: 'member' | 'admin' | 'owner' = 'member'
) {
  const membership = await db.select()
    .from(teamMembers)
    .where(and(
      eq(teamMembers.teamId, teamId),
      eq(teamMembers.userId, userId)
    ))
    .limit(1);

  if (!membership.length) {
    throw new ForbiddenError('Not a team member');
  }

  if (roleHierarchy[membership[0].role] < roleHierarchy[minRole]) {
    throw new ForbiddenError(`Requires ${minRole} role`);
  }

  return membership[0];
}

// Usage in route
app.patch('/teams/:id', requireAuth, async (c) => {
  const teamId = c.req.param('id');
  const session = c.get('session');

  // Only owners can update team settings
  await requireTeamMember(c.env.DB, teamId, session.userId, 'owner');

  // Proceed with update
});
```

## Database Patterns

### Query with Drizzle ORM

```typescript
import { db } from '../lib/db';
import { eq, and, desc } from 'drizzle-orm';
import * as schema from '../db/schema';

// Simple select
const users = await db.select()
  .from(schema.users)
  .where(eq(schema.users.id, userId));

// Join example
const teamsWithMembers = await db.select()
  .from(schema.teams)
  .leftJoin(schema.teamMembers, eq(schema.teams.id, schema.teamMembers.teamId))
  .where(eq(schema.teamMembers.userId, userId));

// Insert with returning
const newUser = await db.insert(schema.users)
  .values({ email, passwordHash, role: 'USER' })
  .returning();

// Update
await db.update(schema.users)
  .set({ lastLoginAt: Date.now() })
  .where(eq(schema.users.id, userId));

// Delete
await db.delete(schema.users)
  .where(eq(schema.users.id, userId));
```

### Transactions

```typescript
await c.env.DB.batch([
  db.insert(schema.teams).values({ domain, ownerId }),
  db.insert(schema.teamMembers).values({ teamId, userId, role: 'owner' }),
]);
```

## Error Handling

### Standard Error Classes

```typescript
import {
  UnauthorizedError,
  ForbiddenError,
  NotFoundError,
  BadRequestError
} from '../lib/errors';

// 401 Unauthorized
throw new UnauthorizedError('Invalid session');

// 403 Forbidden
throw new ForbiddenError('Requires admin role');

// 404 Not Found
throw new NotFoundError('Team not found');

// 400 Bad Request
throw new BadRequestError('Invalid email format');
```

### Error Middleware

Errors are caught by global error middleware:

```typescript
// Automatic error handling in apps/api/src/index.ts
app.onError((err, c) => {
  if (err instanceof UnauthorizedError) {
    return c.json({ error: err.message }, 401);
  }
  if (err instanceof ForbiddenError) {
    return c.json({ error: err.message }, 403);
  }
  // ... etc

  // Unexpected errors → 500
  return c.json({ error: 'Internal server error' }, 500);
});
```

## Validation

Use Zod for request validation:

```typescript
import { z } from 'zod';

const createItemSchema = z.object({
  title: z.string().min(1).max(200),
  description: z.string().optional(),
  priority: z.enum(['P0', 'P1', 'P2', 'P3', 'P4']).default('P2'),
});

app.post('/', requireAuth, async (c) => {
  const body = await c.req.json();

  // Validate
  const parsed = createItemSchema.safeParse(body);
  if (!parsed.success) {
    throw new BadRequestError(parsed.error.message);
  }

  // Use validated data
  const item = await db.insert(schema.items).values(parsed.data);
  return c.json({ item }, 201);
});
```

## CORS Configuration

CORS is configured globally in `apps/api/src/index.ts`:

```typescript
import { cors } from 'hono/cors';

app.use('/*', cors({
  origin: [
    'http://localhost:3000',     // Local dev
    'https://workway.co',         // Production web
    'https://www.workway.co',     // Production web (www)
  ],
  credentials: true,  // Allow cookies
}));
```

## Rate Limiting

Use smart rate limiting per route:

```typescript
import { rateLimiter } from '../middleware/rate-limiter';

// Standard rate limit (100 req/min)
app.get('/items', rateLimiter('standard'), async (c) => {
  // ...
});

// Strict rate limit (10 req/min)
app.post('/items', rateLimiter('strict'), async (c) => {
  // ...
});

// Relaxed rate limit (1000 req/min)
app.get('/public', rateLimiter('relaxed'), async (c) => {
  // ...
});
```

## Response Patterns

### Success Responses

```typescript
// 200 OK (default)
return c.json({ data });

// 201 Created
return c.json({ item }, 201);

// 204 No Content
return c.body(null, 204);
```

### Error Responses

```typescript
// 400 Bad Request
throw new BadRequestError('Invalid input');

// 401 Unauthorized
throw new UnauthorizedError('Session expired');

// 403 Forbidden
throw new ForbiddenError('Requires admin role');

// 404 Not Found
throw new NotFoundError('Item not found');
```

## Testing Patterns

```typescript
// File: apps/api/test/routes/feature.spec.ts
import { describe, it, expect, beforeEach } from 'vitest';
import app from '../../src/index';

describe('Feature API', () => {
  let sessionId: string;

  beforeEach(async () => {
    // Create test session
    const res = await app.request('/auth/login', {
      method: 'POST',
      body: JSON.stringify({ email: 'test@example.com', password: 'test' }),
    });
    sessionId = res.headers.get('set-cookie')!.match(/session_id=([^;]+)/)?.[1]!;
  });

  it('requires authentication', async () => {
    const res = await app.request('/feature');
    expect(res.status).toBe(401);
  });

  it('lists items for authenticated user', async () => {
    const res = await app.request('/feature', {
      headers: { Cookie: `session_id=${sessionId}` },
    });
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body).toHaveProperty('items');
  });
});
```

## Common Patterns

### Pattern 1: CRUD with Team Isolation

```typescript
// List items (team-scoped)
app.get('/', requireAuth, async (c) => {
  const session = c.get('session');

  // Get user's teams
  const memberships = await db.select()
    .from(teamMembers)
    .where(eq(teamMembers.userId, session.userId));

  const teamIds = memberships.map(m => m.teamId);

  // Get items from user's teams
  const items = await db.select()
    .from(schema.items)
    .where(inArray(schema.items.teamId, teamIds));

  return c.json({ items });
});
```

### Pattern 2: OAuth Token Storage

```typescript
// Store OAuth tokens after callback
app.post('/oauth/callback', requireAuth, async (c) => {
  const session = c.get('session');
  const { code, provider } = await c.req.json();

  // Exchange code for tokens
  const tokens = await exchangeCodeForTokens(provider, code);

  // Store in database
  await db.insert(schema.oauthTokens).values({
    userId: session.userId,
    provider,
    accessToken: tokens.access_token,
    refreshToken: tokens.refresh_token,
    expiresAt: Date.now() + tokens.expires_in * 1000,
  });

  return c.json({ success: true });
});
```

### Pattern 3: Pagination

```typescript
app.get('/items', requireAuth, async (c) => {
  const limit = parseInt(c.req.query('limit') || '50');
  const offset = parseInt(c.req.query('offset') || '0');

  const items = await db.select()
    .from(schema.items)
    .limit(limit)
    .offset(offset);

  return c.json({
    items,
    pagination: { limit, offset, hasMore: items.length === limit }
  });
});
```

## Zuhandenheit Test

Before shipping an API endpoint:

1. Does the route name describe the outcome? (`/meetings/follow-up` not `/meetings/process`)
2. Do error messages guide the user? (Not just "Forbidden")
3. Does auth happen transparently? (User doesn't think about sessions)
4. Does team isolation work invisibly? (User doesn't query for their team)

If the mechanism is visible, it's not ready.

## Full Reference

For comprehensive patterns, see:
- [auth-patterns.md](../../.claude/rules/auth-patterns.md) - Domain-based teams, session management
- [deployment.md](../../.claude/rules/deployment.md) - Deployment, rollback, migrations
- [CLAUDE.md](../../CLAUDE.md) - Cross-repo routing, architecture overview
