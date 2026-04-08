---
paths:
  - "**/db/migrate/**"
  - "**/db/schema.rb"
  - "**/db/seeds.rb"
  - "**/app/models/**"
  - "**/*.rb"
---

# Ruby Schema & Data Migration Rules

This variant replaces the global `schema-migration.md` for the kailash-coc-claude-rb USE template. Same principles, ActiveRecord idioms (Sequel projects: substitute `Sequel.migration` and `sequel -M` for the equivalent ActiveRecord patterns).

## MUST Rules

### 1. All Schema Changes Through ActiveRecord Migrations

`CREATE TABLE`, `ADD COLUMN`, `DROP TABLE`, indexes, and any other DDL MUST live in a numbered Rails migration file (`db/migrate/YYYYMMDDHHMMSS_*.rb`). DDL string literals in models, controllers, services, or rake tasks are BLOCKED.

```ruby
# DO — schema lives in a migration
# db/migrate/20260408120000_add_email_index_to_users.rb
class AddEmailIndexToUsers < ActiveRecord::Migration[7.1]
  def change
    add_index :users, :email, unique: true
  end
end

# DO NOT — DDL in application code
class UserService
  def ensure_index
    ActiveRecord::Base.connection.execute("CREATE INDEX idx_users_email ON users(email)")
  end
end
```

**Why:** DDL outside the migration framework runs once on whichever environment touches it and never on the others. The schemas drift, the next deploy fails on the un-migrated environment, and the failure looks like a code bug because the migration was never recorded in `schema_migrations`.

### 2. Data Fixes Are Migrations, Not One-Off Rake Tasks or Console Commands

Backfills, reclassifications, and deduplication MUST be numbered migrations with the same review and rollback discipline as schema changes. Ad-hoc `User.where(...).update_all(...)` in `rails console` or one-off rake tasks against production are BLOCKED.

```ruby
# DO — backfill as a numbered migration
# db/migrate/20260408130000_backfill_user_signup_source.rb
class BackfillUserSignupSource < ActiveRecord::Migration[7.1]
  def up
    User.where(signup_source: nil).update_all(signup_source: 'organic')
  end

  def down
    User.where(signup_source: 'organic').update_all(signup_source: nil)
  end
end

# DO NOT — hotfix in rails console
# rails console> User.where(signup_source: nil).update_all(signup_source: 'organic')
```

**Why:** A console command run by hand has no record, no rollback, and no audit trail. The next environment never gets the same fix, and six months later the team cannot reconstruct why production rows differ from staging.

### 3. Every Migration Has a Reversible Path

Use `change` for fully reversible migrations (Rails detects the inverse), or explicit `up`/`down` methods for non-reversible operations. Migrations whose `down` cannot restore prior state MUST raise `ActiveRecord::IrreversibleMigration` inside `def down` and require human acknowledgement before running. Note: `disable_ddl_transaction!` is a separate flag for non-transactional DDL (e.g., `add_index ..., algorithm: :concurrently`) and is NOT the marker for irreversibility.

```ruby
# DO — change auto-reverses
class AddTierToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :tier, :string, default: 'free'
  end
end

# DO — explicit up/down for non-trivial reverse
class RenameUserStatusToState < ActiveRecord::Migration[7.1]
  def up
    rename_column :users, :status, :state
  end
  def down
    rename_column :users, :state, :status
  end
end

# DO NOT — silent irreversibility
class DropArchivedEvents < ActiveRecord::Migration[7.1]
  def change
    drop_table :archived_events  # data gone, no warning
  end
end
```

**Why:** Migrations are deployed, and deployed code rolls back. Without a reversible path, a failed deploy cannot return to a known-good schema and the system is stuck mid-migration.

### 4. Migration Files Are Append-Only

Once a migration file is committed to a shared branch, it MUST NOT be edited. Mistakes are corrected by adding a new migration that reverses or supersedes the prior one.

**Why:** Editing a committed migration means environments that already ran it have a different schema than environments that run the edited version. The `schema_migrations` table tracking lies, and the drift is undetectable until something breaks.

### 5. Test Migrations Against the Production Adapter

Migration tests MUST run against the same adapter as production (PostgreSQL → PostgreSQL test instance, MySQL → MySQL test instance). SQLite is NOT acceptable for migration validation if production runs PostgreSQL.

```ruby
# config/database.yml — test environment matches production adapter
test:
  adapter: postgresql  # NOT sqlite3 if production is postgresql
  database: myapp_test
```

**Why:** PostgreSQL and SQLite accept different DDL — `BIGSERIAL` vs `INTEGER PRIMARY KEY AUTOINCREMENT`, different index syntax, different constraint behavior. A migration that passes against SQLite can syntax-error against production PostgreSQL on first deploy.

### 6. Production Schema Sync Is a Deploy Gate

`/deploy` MUST verify `rails db:migrate:status` shows zero pending migrations before publishing the new bundle. If migrations are pending, deploy STOPS until reconciled. This check MUST be declared as a gate in `deploy/deployment-config.md` (see `deploy-hygiene.md` § "Pre-deploy gates run before every deploy").

**Why:** Code that assumes a column exists, deployed against a database where the column does not exist yet, throws on first request. The deploy command returns 0; the application is broken; users see errors.

## MUST NOT

- **No "I'll write the migration later" data fixes.** If you change runtime data, you write the migration in the same commit.

**Why:** "Later" means a different session, a different agent, and a high probability of "later" never arriving — production stays patched-by-hand and staging doesn't match.

- **No raw SQL in models or services as a workaround for missing schema.** If the schema doesn't have the column you need, add a migration. Do not coerce the data with `find_by_sql` hacks.

**Why:** Runtime SQL hacks calcify into "the way it works" and the missing schema column never gets added. Two years later, every read of that table has a `CASE WHEN` around the missing column.

- **No `drop_table` / `remove_column` without a preserved-data plan.** Either back the data up to a parking table within the same migration, or explicitly mark the migration as destructive and require human acknowledgement.

**Why:** Dropped data is unrecoverable. A one-line migration mistake during refactor has wiped years of customer history more than once.

- **No bypassing migrations via `psql`/`mysql` shells against production.** All DDL goes through `rails db:migrate`, every time. The `schema_migrations` table is the only ground truth for which migrations have run; manual DDL leaves it out of sync.

**Why:** The framework's tracking table is the only source of truth. Manual DDL leaves it incorrect, and the next automated migration either re-runs or skips changes incorrectly.

## Relationship to Other Rules

- `rules/infrastructure-sql.md` covers query safety (parameterization, dialect portability) inside both application code and migrations.
- `rules/zero-tolerance.md` Rule 4 (no SDK workarounds) applies — if Rails' migration framework is missing a feature, fix it upstream or use the explicit escape hatch with documentation, do not write raw DDL around it.
