---
paths:
  - "lib/**/db/**"
  - "lib/**/infrastructure/**"
---

# Infrastructure SQL Rules

## Scope

These rules apply when editing files in:

- `lib/**/db/**`
- `lib/**/infrastructure/**`

## MUST Rules

### 1. Validate SQL Identifiers

Every method that interpolates a table or column name into SQL MUST validate it first.

```ruby
# DO:
TABLE_NAME_RE = /\A[a-zA-Z_][a-zA-Z0-9_]*\z/

def validate_identifier!(name)
  unless TABLE_NAME_RE.match?(name)
    raise ArgumentError, "Invalid SQL identifier '#{name}': must match [a-zA-Z_][a-zA-Z0-9_]*"
  end
end

validate_identifier!(table_name)
db.execute("SELECT * FROM #{table_name} WHERE id = ?", record_id)

# DO NOT:
db.execute("SELECT * FROM #{user_input} WHERE id = ?", record_id)
```

**Why**: Unvalidated identifiers enable SQL injection. The regex `^[a-zA-Z_][a-zA-Z0-9_]*$` prevents all injection vectors.

**Enforced by**: Red team finding C6 — all public methods in dialect, task queue, and worker registry modules validate identifiers.

### 2. Use Parameterized Queries — Always

All user-provided values MUST use parameterized queries via Sequel or ActiveRecord. NEVER interpolate values into SQL strings.

```ruby
# DO: Sequel parameterized queries
DB[:users].where(id: user_id).first
DB[:users].where(Sequel.like(:name, pattern)).all
DB.fetch("SELECT * FROM users WHERE id = ?", user_id).first

# DO: ActiveRecord parameterized queries
User.where(id: user_id).first
User.where("name LIKE ?", pattern).to_a

# DO NOT: String interpolation
DB.execute("SELECT * FROM users WHERE id = #{user_id}")        # <-- SQL injection!
User.where("name = '#{name}'")                                  # <-- SQL injection!
DB.execute("DELETE FROM users WHERE id = " + id.to_s)           # <-- SQL injection!
```

**Why**: Parameterized queries prevent SQL injection regardless of input content. Ruby's Sequel and ActiveRecord both support parameterized queries natively.

### 3. Use Transactions for Multi-Statement Operations

Any operation involving more than one SQL statement that must be atomic MUST use `DB.transaction`.

```ruby
# DO:
DB.transaction do
  row = DB[:events].where(stream: stream).max(:seq)
  DB[:events].insert(stream: stream, seq: (row || 0) + 1, data: data)
end

# DO NOT:
row = DB[:events].where(stream: stream).max(:seq)
DB[:events].insert(stream: stream, seq: (row || 0) + 1, data: data)
```

**Why**: Without a transaction, auto-commit releases locks between statements. This causes race conditions (C2, C3, C4 in red team report): event store sequence races, idempotency TOCTOU, task queue lock release.

### 4. Use Sequel/ActiveRecord Dialect Abstractions

SQL in infrastructure code MUST use framework abstractions for dialect-portable operations. Do NOT write raw dialect-specific SQL.

```ruby
# DO: Sequel dataset API (dialect-portable)
DB[:tasks].insert(task_id: task_id, status: status)
DB[:tasks].where(task_id: task_id).update(status: "complete")

# DO: When raw SQL is needed, use ? placeholders
DB.fetch("INSERT INTO tasks (task_id, status) VALUES (?, ?)", task_id, status)

# DO NOT: Dialect-specific raw SQL
DB.execute("INSERT INTO tasks VALUES ($1, $2)", task_id, status)  # PostgreSQL only
DB.execute("INSERT INTO tasks VALUES (%s, %s)", task_id, status)  # MySQL only
```

**Why**: Sequel and ActiveRecord translate to dialect-specific format automatically. Hardcoded dialect syntax breaks portability.

### 5. Use Database-Level Upserts, Not Check-Then-Act

Any "insert or update" operation MUST use the database's upsert mechanism.

```ruby
# DO: Sequel upsert (insert_conflict)
DB[:checkpoints].insert_conflict(
  target: [:run_id, :node_id],
  update: { data: data, updated_at: Time.now }
).insert(run_id: run_id, node_id: node_id, data: data, updated_at: Time.now)

# DO: ActiveRecord upsert
Checkpoint.upsert(
  { run_id: run_id, node_id: node_id, data: data, updated_at: Time.now },
  unique_by: [:run_id, :node_id]
)

# DO NOT: Check-then-act (TOCTOU race)
row = DB[:checkpoints].where(run_id: run_id).first
if row
  DB[:checkpoints].where(run_id: run_id).update(data: data)
else
  DB[:checkpoints].insert(run_id: run_id, data: data)
end
```

**Why**: Check-then-act is a TOCTOU race. Between the SELECT and INSERT, another process can insert the same row (M1 in red team report).

### 6. Validate Table Names in Constructors

Store classes that accept a configurable table name MUST validate it in `initialize`.

```ruby
# DO:
TABLE_NAME_RE = /\A[a-zA-Z_][a-zA-Z0-9_]*\z/

def initialize(db, table_name: "kailash_task_queue")
  unless TABLE_NAME_RE.match?(table_name)
    raise ArgumentError, "Invalid table name '#{table_name}': must match [a-zA-Z_][a-zA-Z0-9_]*"
  end
  @db = db
  @table_name = table_name.freeze
end

# DO NOT:
def initialize(db, table_name: "kailash_task_queue")
  @db = db
  @table_name = table_name  # No validation!
end
```

**Why**: Constructor-time validation prevents injection before any SQL is ever generated (C6+ fix in convergence report).

### 7. Bound In-Memory Stores

In-memory stores (Hashes, Arrays) MUST have a maximum size with LRU eviction.

```ruby
# DO:
MAX_ENTRIES = 10_000

class InMemoryExecutionStore
  def initialize(max_entries: MAX_ENTRIES)
    @store = {}
    @order = []
    @max_entries = max_entries
    @mutex = Mutex.new
  end

  def record_start(run_id, data)
    @mutex.synchronize do
      while @store.size >= @max_entries
        oldest = @order.shift
        @store.delete(oldest)
      end
      @order << run_id
      @store[run_id] = data
    end
  end
end

# DO NOT:
class InMemoryExecutionStore
  def initialize
    @store = {}  # Grows without bound -> OOM
  end
end
```

**Why**: Unbounded collections in long-running processes lead to memory exhaustion (H1 in red team report). Default bound: 10,000 entries.

### 8. Lazy Driver Requires

Database driver gems (`pg`, `mysql2`, `sqlite3`) MUST be required inside the method that uses them, not at file top level.

```ruby
# DO:
def init_postgres
  begin
    require "pg"
  rescue LoadError => e
    raise LoadError,
      "pg gem is required for PostgreSQL connections. " \
      "Add it to your Gemfile: gem 'pg'",
      cause: e
  end
  @pool = PG::Connection.new(@url)
end

# DO NOT:
require "pg"  # Top-level require — fails at require time if gem not installed
```

**Why**: `gem install kailash` (without extras) must work at Level 0. Lazy requires ensure optional drivers are only needed when actually used.

## MUST NOT Rules

### 1. No Dialect-Specific DDL in Shared Code

MUST NOT use dialect-specific DDL keywords in table definitions that must work across databases.

```ruby
# DO: Use Sequel migrations (dialect-portable)
DB.create_table(:events) do
  primary_key :id
  String :stream, null: false
  Integer :seq, null: false
  column :data, :text
end

# DO NOT: Hardcoded dialect-specific SQL
DB.execute("CREATE TABLE events (id SERIAL PRIMARY KEY, ...)")     # PostgreSQL only
DB.execute("CREATE TABLE events (id INTEGER AUTOINCREMENT, ...)")  # SQLite only
```

**Why**: Sequel and ActiveRecord migrations abstract dialect differences. Hardcoded DDL breaks portability.

### 2. No Separate Connection Pools Per Store

MUST NOT create a new connection pool for each store instance.

```ruby
# DO:
factory = Kailash::StoreFactory.default
factory.initialize!
# All stores share factory.db internally

# DO NOT:
conn1 = Sequel.connect("postgres://...")
conn2 = Sequel.connect("postgres://...")
event_store  = DBEventStore.new(conn1)
exec_store   = DBExecutionStore.new(conn2)
```

**Why**: Each connection creates its own pool. Multiple pools to the same database waste connections and prevent transaction isolation across stores.

### 3. No `FOR UPDATE SKIP LOCKED` Without a Transaction

MUST NOT use `FOR UPDATE SKIP LOCKED` outside of an explicit transaction.

```ruby
# DO:
DB.transaction do
  row = DB[:tasks]
    .where(status: "pending")
    .order(:created_at)
    .limit(1)
    .for_update
    .skip_locked
    .first
  DB[:tasks].where(task_id: row[:task_id]).update(status: "processing") if row
end

# DO NOT:
row = DB[:tasks]
  .where(status: "pending")
  .for_update
  .skip_locked
  .first
# Lock is released immediately on auto-commit! Another worker can claim the same row.
DB[:tasks].where(task_id: row[:task_id]).update(status: "processing")
```

**Why**: `FOR UPDATE SKIP LOCKED` acquires a row-level lock. In auto-commit mode, the lock is released as soon as the SELECT completes. The subsequent UPDATE then races with other workers (C4 in red team report).

## Cross-References

- `lib/**/db/dialect.rb` — QueryDialect, `validate_identifier!`, `validate_json_path!`
- `lib/**/db/connection.rb` — ConnectionManager, transaction proxy
- `lib/**/infrastructure/factory.rb` — StoreFactory singleton
- `.claude/rules/security.md` — Global security rules (parameterized queries, no secrets)
