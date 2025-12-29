---
name: database-migrations
description: Safe D1 database migration patterns for WORKWAY platform. Apply when creating schema changes, handling SQLite constraints, or planning rollback strategies. Embodies "thorough" from Rams' principles.
---

# Database Migration Patterns

Safe database migration practices for WORKWAY platform (D1/SQLite).

## Philosophy: Thoroughness

**Every detail matters.** Database migrations are irreversible by default. A missed constraint or wrong data type can corrupt production data. This skill embodies Rams' 8th principle: "Good design is thorough."

## Core Constraints

### D1 is SQLite

D1 is built on SQLite. These constraints differ from PostgreSQL/MySQL:

| Operation | Support | Workaround |
|-----------|---------|------------|
| `DROP COLUMN` | Not supported | Rebuild table |
| `RENAME COLUMN` | Supported (SQLite 3.25+) | Use `ALTER TABLE ... RENAME COLUMN` |
| `ALTER COLUMN TYPE` | Not supported | Rebuild table |
| `ADD CONSTRAINT` (after creation) | Not supported | Rebuild table |
| `DROP CONSTRAINT` | Not supported | Rebuild table |
| Multiple `ALTER TABLE` in one statement | Not supported | Separate statements |

**Table rebuild is risky** - locks table, can fail mid-migration. Always backup first.

## Migration Classification

Before creating a migration, classify it:

| Type | Examples | Rollback Complexity | Risk Level |
|------|----------|-------------------|------------|
| **Additive** | CREATE TABLE, ADD COLUMN, CREATE INDEX | Low | Low |
| **Destructive** | DROP TABLE, DROP COLUMN, DELETE data | **Irreversible** | High |
| **Transformative** | Rename column, modify constraints, data migration | Medium | Medium |

## Pre-Flight Checklist

Complete ALL items before applying to production:

### 1. Classification
- [ ] Migration type identified (additive/destructive/transformative)
- [ ] Risk level assessed

### 2. Local Testing
- [ ] Migration tested locally: `wrangler d1 migrations apply marketplace-production --local`
- [ ] SQL syntax verified (no SQLite gotchas)
- [ ] Migration file named correctly: `NNNN_descriptive_name.sql`

### 3. Data Safety
- [ ] Backup critical tables (if destructive):
  ```bash
  wrangler d1 execute marketplace-production --remote \
    --command "SELECT * FROM table_name" > backup_table_name.json
  ```
- [ ] Row count recorded before migration

### 4. Rollback Preparation
- [ ] Down migration written (for non-additive changes)
- [ ] Down migration tested locally
- [ ] Rollback documented in migration file header

### 5. Deployment Timing
- [ ] Low-traffic window scheduled (avoid Mon-Fri 9am-5pm user timezone)
- [ ] Team notified in #engineering
- [ ] Cloudflare D1 dashboard open for monitoring

### 6. Post-Migration Verification
- [ ] Migration applied: `wrangler d1 migrations list marketplace-production --remote`
- [ ] Schema correct: Verify with `PRAGMA table_info(table_name)`
- [ ] Application working: Test affected endpoints
- [ ] No error spikes: Check Workers & Pages → workway-api → Metrics for 5 minutes

## Migration File Format

### Naming Convention

Format: `NNNN_descriptive_action.sql`

Use sequential 4-digit prefix (0057, 0058, etc.)

**Examples:**
- `0057_add_team_billing_columns.sql` (good - describes action)
- `0057_teams.sql` (too vague)
- `0057_add_stripe_subscription_id_to_teams.sql` (good - specific)

### Header Template

```sql
-- Migration: 0057_add_team_billing_columns
-- Description: Add Stripe billing columns to teams table
-- Type: Additive (low risk)
-- Rollback: Not required (unused columns harmless) OR See down migration
-- ============================================================================

-- Add billing-related columns
ALTER TABLE teams ADD COLUMN stripe_subscription_id TEXT;
ALTER TABLE teams ADD COLUMN billing_email TEXT;

-- Index for billing lookups
CREATE INDEX idx_teams_stripe ON teams(stripe_subscription_id);
```

## Common Migration Patterns

### Pattern 1: Add Column (Additive)

```sql
-- Migration: 0058_add_users_last_login
-- Type: Additive
-- Rollback: Not required
-- ============================================================================

ALTER TABLE users ADD COLUMN last_login_at INTEGER;
```

**Rollback:** Not strictly necessary (unused column is harmless), but if needed, use table rebuild pattern.

### Pattern 2: Create Table (Additive)

```sql
-- Migration: 0059_create_audit_logs
-- Type: Additive
-- Rollback: DROP TABLE audit_logs
-- ============================================================================

CREATE TABLE audit_logs (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  action TEXT NOT NULL,
  resource_type TEXT NOT NULL,
  resource_id TEXT,
  metadata TEXT,  -- JSON
  created_at INTEGER DEFAULT (unixepoch()),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_audit_logs_user ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
```

**Rollback:**
```sql
DROP TABLE audit_logs;
```

### Pattern 3: Rename Column (Transformative)

```sql
-- Migration: 0060_rename_workflow_runs_old_name
-- Type: Transformative
-- Rollback: ALTER TABLE workflow_runs RENAME COLUMN new_name TO old_name
-- ============================================================================

ALTER TABLE workflow_runs RENAME COLUMN old_name TO new_name;
```

**Rollback:** Reverse rename
```sql
ALTER TABLE workflow_runs RENAME COLUMN new_name TO old_name;
```

**Critical:** Code must be deployed with migration. Rollback requires reverting code first, then reverting migration.

### Pattern 4: Table Rebuild (High Risk)

When you need to drop a column or change a column type:

```sql
-- Migration: 0061_rebuild_teams_remove_deprecated
-- Type: Destructive
-- Rollback: Restore from backup (manual)
-- CRITICAL: Backup required before applying
-- ============================================================================

-- Step 1: Create new table with desired schema
CREATE TABLE teams_new (
  id TEXT PRIMARY KEY,
  domain TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  owner_id TEXT NOT NULL REFERENCES users(id),
  stripe_customer_id TEXT,
  created_at INTEGER DEFAULT (unixepoch()),
  updated_at INTEGER DEFAULT (unixepoch())
  -- NOTE: deprecated_column is NOT included
);

-- Step 2: Copy data (map old columns to new)
INSERT INTO teams_new (id, domain, name, owner_id, stripe_customer_id, created_at, updated_at)
SELECT id, domain, name, owner_id, stripe_customer_id, created_at, updated_at
FROM teams;

-- Step 3: Drop old table
DROP TABLE teams;

-- Step 4: Rename new table
ALTER TABLE teams_new RENAME TO teams;

-- Step 5: Recreate indexes
CREATE INDEX idx_teams_domain ON teams(domain);
CREATE INDEX idx_teams_owner ON teams(owner_id);
```

**Rollback:** IMPOSSIBLE without backup. Must export data before migration:
```bash
wrangler d1 execute marketplace-production --remote \
  --command "SELECT * FROM teams" > backup_teams_before_0061.json
```

### Pattern 5: Data Migration (Transformative)

```sql
-- Migration: 0062_migrate_role_format
-- Type: Transformative
-- Rollback: Reverse UPDATE
-- ============================================================================

-- Migrate old role format to new format
UPDATE team_members SET role = 'owner' WHERE role = 'OWNER';
UPDATE team_members SET role = 'admin' WHERE role = 'ADMIN';
UPDATE team_members SET role = 'member' WHERE role = 'MEMBER';
```

**Rollback:**
```sql
UPDATE team_members SET role = 'OWNER' WHERE role = 'owner';
UPDATE team_members SET role = 'ADMIN' WHERE role = 'admin';
UPDATE team_members SET role = 'MEMBER' WHERE role = 'member';
```

## Rollback Strategies

### Deploy Order

When a migration and code change are deployed together:

**Forward Deployment:**
1. Apply database migration first
2. Then deploy Worker code

**Why?** Old code can handle missing columns (with optional chaining). New code cannot handle missing columns.

**Rollback Order (reverse):**
1. Rollback Worker code first
2. Then apply down migration (if safe)

### Rollback Decision Matrix

| Scenario | Action |
|----------|--------|
| Additive migration, no issues | No rollback needed |
| Additive migration, causes errors | Rollback code first, then decide on migration |
| Destructive migration, success | Cannot rollback (by design) |
| Destructive migration, wrong data deleted | Restore from backup |
| Transformative migration, issues | Apply down migration, then rollback code |

## Emergency Procedures

### Migration Failed Mid-Apply

1. Check migration status:
   ```bash
   wrangler d1 migrations list marketplace-production --remote
   ```

2. Check table state:
   ```bash
   wrangler d1 execute marketplace-production --remote --command ".schema table_name"
   ```

3. If partial state, may need manual cleanup (documented case-by-case)

### Data Corruption Detected

1. **Stop the API immediately** to prevent further damage
   ```bash
   # In workway-platform/apps/api
   wrangler rollback --name workway-api --version <previous-version>
   ```

2. Assess scope: which tables, how many rows

3. If backup exists: restore from backup

4. If no backup: check if data exists in application logs or external systems

5. Post-mortem: why wasn't this caught in testing?

## Commands Reference

```bash
# Create a new migration
wrangler d1 migrations create marketplace-production add_feature_x

# Test locally first (ALWAYS do this)
wrangler d1 migrations apply marketplace-production --local

# Apply to production
wrangler d1 migrations apply marketplace-production --remote

# Check migration status
wrangler d1 migrations list marketplace-production --remote

# Execute raw SQL (use with extreme caution)
wrangler d1 execute marketplace-production --remote --command="SELECT COUNT(*) FROM users"

# Backup table before destructive change
wrangler d1 execute marketplace-production --remote \
  --command "SELECT * FROM table_name" > backup_table_name.json
```

## Testing Migrations Locally

```bash
# 1. Create local D1 database (if not exists)
wrangler d1 create marketplace-production --local

# 2. Apply migrations to local database
wrangler d1 migrations apply marketplace-production --local

# 3. Test migration with sample data
wrangler d1 execute marketplace-production --local --command="
  INSERT INTO users (id, email, role) VALUES ('test-1', 'test@example.com', 'USER');
  SELECT * FROM users;
"

# 4. Verify schema
wrangler d1 execute marketplace-production --local --command=".schema users"

# 5. Clean up test data
wrangler d1 execute marketplace-production --local --command="DELETE FROM users WHERE id = 'test-1'"
```

## Schema Verification

After applying a migration, verify the schema matches expectations:

```bash
# View full schema
wrangler d1 execute marketplace-production --remote --command=".schema"

# View specific table
wrangler d1 execute marketplace-production --remote --command=".schema users"

# View table info (columns, types)
wrangler d1 execute marketplace-production --remote --command="PRAGMA table_info(users)"

# View indexes
wrangler d1 execute marketplace-production --remote --command="PRAGMA index_list(users)"
```

## Monitoring After Migration

1. **Check error rates** (Workers & Pages → workway-api → Metrics)
   - Should remain < 0.1% error rate

2. **Test affected endpoints**
   ```bash
   curl https://api.workway.co/health
   curl -H "Authorization: Bearer $TOKEN" https://api.workway.co/users/me
   ```

3. **Verify data integrity**
   ```bash
   wrangler d1 execute marketplace-production --remote \
     --command "SELECT COUNT(*) FROM table_name"
   ```

4. **Monitor for 5-10 minutes** before considering migration complete

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|--------------|----------------|------------------|
| Skip local testing | Production failures | Always test locally first |
| No backup before destructive change | Data loss | Export data before migration |
| Multiple schema changes in one migration | Hard to rollback | One logical change per migration |
| No header comments | Context loss | Document type, rollback, rationale |
| Deploy code before migration | App expects schema that doesn't exist | Migration first, code second |

## Zuhandenheit Test

Migrations should be thorough but invisible to users:

1. Does the migration run without user-visible downtime?
2. If migration fails, can we rollback without data loss?
3. Are error messages (if any) helpful to engineers?
4. Does the schema change support the outcome (not just the implementation)?

If users notice the migration, it's not ready.

## Full Reference

For comprehensive deployment context:
- [deployment.md](../../.claude/rules/deployment.md) - Full deployment safety, rollback procedures
- [auth-patterns.md](../../.claude/rules/auth-patterns.md) - Schema patterns for teams, sessions
- [CLAUDE.md](../../CLAUDE.md) - Cross-repo coordination
