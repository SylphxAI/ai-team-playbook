# Database Migration Strategy

> Parallel agents and sequential migration files are fundamentally incompatible.
> This document explains why and what to do about it.

## The Problem

Traditional migration tools use sequential numbering:

```
migrations/
  0001_create_users.sql
  0002_add_email_to_users.sql
  0003_create_posts.sql
  0004_add_published_to_posts.sql
```

This works when one developer creates migrations sequentially. It breaks catastrophically with parallel AI agents:

```
Timeline:
  t=0: main has migrations 0001-0003
  t=1: Agent A branches, creates 0004_add_comments.sql
  t=2: Agent B branches, creates 0004_add_likes.sql
  t=3: Agent A's PR passes CI (tests against DB with 0001-0003 + 0004_comments)
  t=4: Agent B's PR passes CI (tests against DB with 0001-0003 + 0004_likes)
  t=5: Agent A merges → main has 0001-0004 (comments)
  t=6: Agent B merges → CONFLICT: two files named 0004_*
```

Even if you avoid the naming conflict (Drizzle uses hashes), the migration runner might:
- Apply them in the wrong order
- Skip one because it thinks it was already applied
- Apply both but in an order that violates foreign key constraints

## Drizzle ORM Limitations

Drizzle's `drizzle-kit generate` creates migration files with sequential numbering and a `_journal.json` that tracks order:

```json
{
  "entries": [
    { "idx": 0, "when": 1707000000, "tag": "0000_initial" },
    { "idx": 1, "when": 1707100000, "tag": "0001_add_users" },
    { "idx": 2, "when": 1707200000, "tag": "0002_add_posts" }
  ]
}
```

Two agents generating migrations will both create `idx: 3` entries. When merged, the journal has duplicate indices.

**Drizzle's `migrate()` function**:
- Reads the journal
- Compares against the `__drizzle_migrations` table
- Applies missing migrations in order

With duplicate indices, behavior is undefined. We've seen it:
- Skip the second migration entirely
- Apply both but mark only one as applied
- Throw an error and halt the application on startup

## Solution 1: Timestamp-Based Naming

Instead of sequential numbers, use timestamps:

```
migrations/
  20260210_120000_create_users.sql
  20260210_143000_add_email.sql
  20260211_091500_create_posts.sql
  20260212_100000_add_comments.sql   ← Agent A
  20260212_100100_add_likes.sql      ← Agent B (different timestamp)
```

**Pros**: No naming conflicts. Natural ordering.
**Cons**: Drizzle doesn't support this natively. You'd need a custom migration runner.

## Solution 2: Atlas (Recommended)

[Atlas](https://atlasgo.io/) is a declarative migration tool. Instead of writing migration files, you declare the desired schema:

```hcl
// schema.hcl
table "users" {
  schema = schema.public
  column "id" {
    type = uuid
    default = sql("gen_random_uuid()")
  }
  column "email" {
    type = varchar(255)
  }
  column "created_at" {
    type = timestamptz
    default = sql("now()")
  }
  primary_key {
    columns = [column.id]
  }
}
```

Atlas compares the declared schema against the actual database and generates the required SQL:

```bash
atlas schema diff \
  --from "postgresql://localhost:5432/mydb" \
  --to "file://schema.hcl" \
  --dev-url "docker://postgres/16"
```

Output:
```sql
ALTER TABLE "users" ADD COLUMN "email" varchar(255);
```

**Why this works with parallel agents:**
- Agents modify the schema declaration, not migration files
- Two agents can independently add columns to the same schema file
- Git merge handles the text merge (or flags a conflict)
- Atlas generates the correct migration from the merged schema

## Solution 3: drizzle-kit push (Development)

For development/staging, skip migration files entirely:

```bash
drizzle-kit push
```

This directly syncs your Drizzle schema to the database without creating migration files. It:
- Compares your TypeScript schema against the database
- Generates and applies ALTER statements directly
- No migration files, no journal, no conflicts

**Use for**: Development, staging, pre-production environments.
**Don't use for**: Production (no migration history, no rollback).

## The Transition Path

Here's how we migrated from Drizzle migrations to Atlas:

### Phase 1: Stop Creating Migrations (Immediate)

```typescript
// drizzle.config.ts
export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  // Remove: migrations: { ... }
  // Agents use `drizzle-kit push` for development
});
```

Update agent prompts: "Do NOT run `drizzle-kit generate`. Use `drizzle-kit push` to sync schema changes."

### Phase 2: Set Up Atlas (Week 1)

```bash
# Install Atlas
curl -sSf https://atlasgo.sh | sh

# Create atlas.hcl config
cat > atlas.hcl << EOF
env "local" {
  src = "file://schema.hcl"
  url = "postgresql://localhost:5432/mydb?sslmode=disable"
  dev = "docker://postgres/16/dev"
  
  migration {
    dir = "file://migrations"
  }
}
EOF
```

### Phase 3: Generate Baseline Migration

```bash
# Capture current database state as the baseline
atlas migrate diff baseline \
  --env local \
  --to "postgresql://localhost:5432/mydb"
```

### Phase 4: Use Atlas for All Future Migrations

```bash
# When schema changes:
atlas migrate diff add_likes_table --env local

# In CI:
atlas migrate lint --env local --latest 1

# In production:
atlas migrate apply --env local
```

## Schema as Code (Drizzle + Atlas)

You can keep Drizzle for queries and type safety while using Atlas for migrations:

```typescript
// src/db/schema.ts — Drizzle schema (used for queries)
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull(),
  createdAt: timestamp('created_at').defaultNow(),
});
```

```hcl
// schema.hcl — Atlas schema (used for migrations)
// Keep in sync with Drizzle schema
table "users" {
  schema = schema.public
  column "id" { type = uuid; default = sql("gen_random_uuid()") }
  column "email" { type = varchar(255); null = false }
  column "created_at" { type = timestamptz; default = sql("now()") }
  primary_key { columns = [column.id] }
}
```

**Keeping them in sync**: This is the main challenge. Options:
1. Manual sync (error-prone but simple)
2. Generate Atlas schema from Drizzle schema (custom script)
3. Use Atlas's [Drizzle provider](https://atlasgo.io/guides/orms/drizzle) to read Drizzle schema directly

## CI Integration

```yaml
# .github/workflows/migration-check.yml
name: Migration Check

on:
  pull_request:
    paths:
      - 'src/db/schema.ts'
      - 'schema.hcl'
      - 'migrations/**'

jobs:
  check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      
      - uses: ariga/setup-atlas@v1
      
      - name: Lint migrations
        run: atlas migrate lint --env ci --latest 1
      
      - name: Verify migration applies cleanly
        run: atlas migrate apply --env ci --dry-run
```

This catches:
- Destructive changes (DROP TABLE without explicit approval)
- Invalid SQL
- Migrations that don't apply cleanly on a fresh database
- Schema drift between Drizzle and Atlas definitions
