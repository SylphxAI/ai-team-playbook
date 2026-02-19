# Atlas Migrations

> Use Atlas for declarative schema management instead of Drizzle's sequential migration files to avoid conflicts in parallel agent workflows.

## Problem

Drizzle generates sequential migration files. When multiple agents work in parallel and each makes schema changes, they create conflicting migration files that can't be merged cleanly.

Agent A creates `0003_add_users_index.sql`. Agent B creates `0003_add_posts_table.sql`. Now you have two files with the same sequence number, and applying them in either order may break the other.

## Solution

Atlas uses a **declarative schema** — you define what the schema should look like, and Atlas generates the correct diff migration regardless of who runs it or in what order.

Keep Drizzle ORM for querying. Use Atlas only for migration management.

**Why Atlas over Drizzle migrations:**
- Drizzle: generates sequential migration files → parallel agents create conflicts
- Atlas: declarative schema → generates correct diff regardless of who runs it

## Implementation

### Install Atlas

```bash
curl -sSf https://atlasgo.sh | sh
```

### Config: `atlas.hcl` at repo root

```hcl
env "local" {
  src = "file://drizzle/schema.ts"
  dev = "postgres://localhost:5432/dev?sslmode=disable"
  migration {
    dir = "file://migrations"
  }
}
```

### Create a Migration

```bash
bunx atlas migrate diff <name> \
  --to file://drizzle/schema.ts \
  --dev-url "postgres://localhost:5432/dev?sslmode=disable"
```

This inspects the current state of your dev DB, diffs it against the schema definition, and generates only what's needed.

### Apply Migrations

```bash
bunx atlas migrate apply --url "$DATABASE_URL" --dir file://migrations
```

### CI Integration

Add to your CI pipeline after running tests:

```yaml
- name: Apply migrations
  run: bunx atlas migrate apply --url "$DATABASE_URL" --dir file://migrations
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

## Examples

**Scenario: Two agents make schema changes simultaneously**

Agent A adds a `users` index. Agent B adds a `posts` table.

With Drizzle (broken):
```
migrations/0003_add_users_index.sql   ← Agent A
migrations/0003_add_posts_table.sql   ← Agent B (conflict!)
```

With Atlas (works):
```
migrations/20240115_add_users_index.sql  ← timestamped, no conflict
migrations/20240115_add_posts_table.sql  ← Atlas applies both correctly
```

Atlas generates the migration from the schema diff, not from a sequence number. Both can be applied in any order.

## Lessons & Gotchas

- **Keep Drizzle for querying, Atlas for migrations.** Don't rip out Drizzle — it's excellent ORM. Atlas only replaces the migration generation/application step.
- **The dev URL is not your production DB.** Atlas needs a "dev" database to simulate schema diffs. This should be a throwaway local or CI database, not your real dev branch.
- **Atlas reads the schema from source.** Point `--to` at your Drizzle schema file, not a migration. Atlas computes what SQL is needed to get from current state to desired state.
- **Run `atlas migrate apply` in CI, not locally.** Local applies lead to drift between environments. CI apply ensures every environment gets the same migrations in the same order.
- **Atlas lint catches issues before apply.** Run `bunx atlas migrate lint` in CI to catch unsafe operations (e.g., dropping columns that still have data) before they reach production.
