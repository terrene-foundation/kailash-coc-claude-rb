---
paths:
  - "lib/**/db/**"
  - "lib/**/infrastructure/**"
---

# Infrastructure SQL Rules

### 1. Validate SQL Identifiers

```ruby
# DO:
TABLE_NAME_RE = /\A[a-zA-Z_][a-zA-Z0-9_]*\z/

def validate_identifier!(name)
  raise ArgumentError, "Invalid SQL identifier" unless TABLE_NAME_RE.match?(name)
end

validate_identifier!(table_name)
db.execute("SELECT * FROM #{table_name} WHERE id = ?", record_id)

# DO NOT:
db.execute("SELECT * FROM #{user_input} WHERE id = ?", record_id)
```

**Why:** Unvalidated identifiers in string interpolation allow table/column injection even when values are parameterized.

### 2. Parameterized Queries — Always

```ruby
# DO:
DB[:users].where(id: user_id).first
DB.fetch("SELECT * FROM users WHERE id = ?", user_id).first

# DO NOT:
DB.execute("SELECT * FROM users WHERE id = #{user_id}")
```

**Why:** Ruby's `#{}` interpolation inlines raw input into SQL strings, making every unparameterized query a direct injection vector.

### 3. Use Transactions for Multi-Statement Operations

```ruby
# DO:
DB.transaction do
  row = DB[:events].where(stream: stream).max(:seq)
  DB[:events].insert(stream: stream, seq: (row || 0) + 1, data: data)
end

# DO NOT (auto-commit releases locks between statements — race conditions):
row = DB[:events].where(stream: stream).max(:seq)
DB[:events].insert(stream: stream, seq: (row || 0) + 1, data: data)
```

**Why:** Without a transaction block, Sequel auto-commits after each statement, releasing locks between read and write and creating race conditions under concurrency.

### 4. Use Sequel/ActiveRecord Dialect Abstractions

```ruby
# DO:
DB[:tasks].insert(task_id: task_id, status: status)
DB.fetch("INSERT INTO tasks (task_id, status) VALUES (?, ?)", task_id, status)

# DO NOT:
DB.execute("INSERT INTO tasks VALUES ($1, $2)", task_id, status)  # PostgreSQL only
```

**Why:** Dialect-specific placeholders (`$1`) silently fail or produce wrong results when the app runs against SQLite or MySQL in development or testing.

### 5. Use Database-Level Upserts, Not Check-Then-Act

```ruby
# DO:
DB[:checkpoints].insert_conflict(
  target: [:run_id, :node_id],
  update: { data: data, updated_at: Time.now }
).insert(run_id: run_id, node_id: node_id, data: data)

# DO NOT (TOCTOU race):
row = DB[:checkpoints].where(run_id: run_id).first
if row then DB[:checkpoints].where(run_id: run_id).update(data: data)
else DB[:checkpoints].insert(run_id: run_id, data: data) end
```

**Why:** Check-then-act has a TOCTOU window where concurrent requests both see "not found" and both insert, causing unique constraint violations or duplicate records.

### 6. Validate Table Names in Constructors

```ruby
# DO:
def initialize(db, table_name: "kailash_task_queue")
  raise ArgumentError unless /\A[a-zA-Z_][a-zA-Z0-9_]*\z/.match?(table_name)
  @table_name = table_name.freeze
end
```

**Why:** A table name validated once at construction and frozen prevents injection for the lifetime of the object, rather than checking on every query.

### 7. Bound In-Memory Stores

Default bound: 10,000 entries with LRU eviction.

```ruby
# DO:
while @store.size >= @max_entries
  oldest = @order.shift
  @store.delete(oldest)
end

# DO NOT:
@store = {}  # Grows without bound -> OOM
```

**Why:** Ruby Hashes grow without limit and are never compacted by GC while referenced, so an unbounded store causes OOM in long-running processes.

### 8. Lazy Driver Requires

```ruby
# DO:
def init_postgres
  require "pg"
rescue LoadError => e
  raise LoadError, "pg gem required for PostgreSQL. Add to Gemfile.", cause: e
end

# DO NOT:
require "pg"  # Top-level — fails at require time if not installed
```

**Why:** Top-level `require` forces every Bundler environment to include the `pg` gem even when using SQLite, breaking `bundle install` on systems without libpq headers.

## MUST NOT

- **No dialect-specific DDL** in shared code — use Sequel/ActiveRecord migrations
  **Why:** Dialect-specific DDL silently breaks when switching databases (e.g., PostgreSQL `SERIAL` does not exist in SQLite), causing migration failures in dev/test environments.
- **No separate connection pools per store** — share via `StoreFactory.default`
  **Why:** Each Sequel connection pool reserves its own set of database connections, so multiple pools multiply connection usage and exhaust `max_connections`.
- **No `FOR UPDATE SKIP LOCKED` without transaction** — lock releases on auto-commit
  **Why:** Without a wrapping transaction, the row lock is released immediately after the SELECT returns, providing zero concurrency protection.
