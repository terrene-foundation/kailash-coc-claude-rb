---
paths:
  - "**/*.rb"
---

# Connection Pool Safety Rules

## Scope

These rules apply to all code that uses DataFlow, creates database connections, or configures application workers.

**Applies to**: `**/*.rb`

## MUST Rules

### 1. Never Use Default Pool Size in Production

MUST set `DATAFLOW_MAX_CONNECTIONS` environment variable. The default pool size (typically 25 per worker) will exhaust PostgreSQL's connection limit on small-to-medium instances.

**Formula**: `pool_size = postgres_max_connections / num_workers * 0.7`

| Instance Size | `max_connections` | Workers | `DATAFLOW_MAX_CONNECTIONS` |
| ------------- | ----------------- | ------- | -------------------------- |
| t2.micro      | 87                | 2       | 30                         |
| t2.small      | 150               | 2       | 50                         |
| t2.medium     | 150               | 2       | 50                         |
| t3.medium     | 150               | 4       | 25                         |
| r5.large      | 1000              | 4       | 175                        |

```ruby
# WRONG -- relies on default pool size
config = Kailash::DataFlow::Config.new("postgresql://...")
df = Kailash::DataFlow.new(config)

# RIGHT -- explicit pool size from environment
config = Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL"))
config.max_connections = ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i
df = Kailash::DataFlow.new(config)
```

**With ConnectionPool gem:**

```ruby
# RIGHT -- explicit pool size with connection_pool gem
require "connection_pool"

POOL = ConnectionPool.new(
  size: ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i,
  timeout: 5
) do
  PG.connect(ENV.fetch("DATABASE_URL"))
end
```

**With Sequel:**

```ruby
# RIGHT -- explicit pool size with Sequel
DB = Sequel.connect(
  ENV.fetch("DATABASE_URL"),
  max_connections: ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i,
  pool_timeout: 5
)
```

**Enforced by**: deployment-specialist agent, pool-safety skill
**Violation**: BLOCK deployment

### 2. Never Query DB Per-Request in Middleware

MUST NOT call DataFlow workflows (ReadUser, ReadSession, etc.) in middleware that runs on every HTTP request. This creates N+1 connection usage where N is concurrent requests, rapidly exhausting the pool.

**Detection patterns:**

```ruby
# WRONG -- DB query runs on EVERY request (Rack middleware)
class AuthMiddleware
  def call(env)
    Kailash::Registry.open do |registry|
      builder = Kailash::WorkflowBuilder.new
      builder.add_node("ReadUser", "read", { "token" => token })
      workflow = builder.build(registry)
      Kailash::Runtime.open(registry) do |rt|
        result = rt.execute(workflow, {})  # Pool checkout per request!
      end
    end
  end
end

# WRONG -- ActiveRecord query in middleware
class AuthMiddleware
  def call(env)
    user = User.find_by(token: env["HTTP_AUTHORIZATION"])  # DB hit per request!
    env["current_user"] = user
    @app.call(env)
  end
end
```

**Correct patterns:**

```ruby
# RIGHT -- JWT claims, no DB hit
class AuthMiddleware
  def call(env)
    token = env["HTTP_AUTHORIZATION"]&.delete_prefix("Bearer ")
    claims = JWT.decode(token, ENV.fetch("JWT_SECRET"), true, algorithm: "HS256").first
    env["user_id"] = claims["sub"]
    env["permissions"] = claims.fetch("permissions", [])
    @app.call(env)
  end
end

# RIGHT -- In-memory cache with TTL for session data
require "lru_redux"  # or use concurrent-ruby's TimerTask

class AuthMiddleware
  SESSION_CACHE = LruRedux::TTL::ThreadSafeCache.new(1000, 300)  # 1000 entries, 5-min TTL

  def call(env)
    token = extract_token(env)
    user = SESSION_CACHE.getset(token) { fetch_user_from_db(token) }
    env["current_user"] = user
    @app.call(env)
  end
end
```

**Enforced by**: intermediate-reviewer agent, pool-safety skill
**Violation**: BLOCK commit

### 3. Health Checks Must Not Use Application Pool

MUST use a lightweight query or a dedicated connection for health checks, never a full DataFlow workflow. Health checks run every 10-30 seconds per load balancer target -- if they use the application pool, they consume connections continuously.

```ruby
# WRONG -- full DataFlow workflow for health check
get "/health" do
  builder = Kailash::WorkflowBuilder.new
  builder.add_node("ListUser", "check", { "limit" => 1 })
  # ...

# RIGHT -- lightweight raw query, dedicated connection
get "/health" do
  ActiveRecord::Base.connection.execute("SELECT 1")
  { status: "ok" }.to_json
rescue StandardError => e
  status 503
  { status: "error", detail: e.message }.to_json
end

# RIGHT -- dedicated PG connection (not from pool)
get "/health" do
  conn = PG.connect(ENV.fetch("DATABASE_URL"))
  conn.exec("SELECT 1")
  conn.close
  { status: "ok" }.to_json
rescue StandardError => e
  status 503
  { status: "error", detail: e.message }.to_json
end
```

**Enforced by**: deployment-specialist agent
**Violation**: BLOCK deployment

### 4. Verify Pool Math at Deployment

Before deploying, MUST verify this inequality holds:

```
DATAFLOW_MAX_CONNECTIONS x num_workers <= postgres_max_connections x 0.7
```

The 0.7 factor reserves 30% of connections for admin tasks, migrations, monitoring, and connection spikes.

**Example**:

- PostgreSQL `max_connections = 150` (t2.medium default)
- Puma workers = 4
- `DATAFLOW_MAX_CONNECTIONS = 25`
- Check: `25 x 4 = 100 <= 150 x 0.7 = 105` -- PASS

**Counter-example** (the bug that created this rule):

- PostgreSQL `max_connections = 150`
- Puma workers = 4
- `DATAFLOW_MAX_CONNECTIONS = 50` (or default)
- Check: `50 x 4 = 200 > 105` -- FAIL. Pool exhaustion under load.

**Enforced by**: pool-safety skill during `/deploy`
**Violation**: BLOCK deployment

### 5. Connection Timeout Must Be Set

MUST set a connection acquisition timeout. Without it, requests queue indefinitely when the pool is exhausted, causing cascading timeouts.

```ruby
# WRONG -- no timeout, requests hang forever
config = Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL"))
df = Kailash::DataFlow.new(config)

# RIGHT -- timeout after 5 seconds
config = Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL"))
config.max_connections = ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i
config.connection_timeout = 5  # seconds
df = Kailash::DataFlow.new(config)

# RIGHT -- with ConnectionPool gem
POOL = ConnectionPool.new(size: 10, timeout: 5) { PG.connect(ENV.fetch("DATABASE_URL")) }

# RIGHT -- with Sequel
DB = Sequel.connect(ENV.fetch("DATABASE_URL"), max_connections: 10, pool_timeout: 5)
```

### 6. Workers Must Share Pool

All threads within a single Puma/Unicorn worker MUST share one pool. MUST NOT create a new DataFlow/pool instance per request or per route handler.

```ruby
# WRONG -- new connection per request
get "/users" do
  config = Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL"))
  df = Kailash::DataFlow.new(config)  # New pool!
  # ...
end

# RIGHT -- shared at application level (Sinatra)
configure do
  set :df, Kailash::DataFlow.new(
    Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL")).tap { |c|
      c.max_connections = ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i
    }
  )
end

get "/users" do
  settings.df  # Shared pool
  # ...
end

# RIGHT -- shared at application level (Rails initializer)
# config/initializers/kailash.rb
config = Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL"))
config.max_connections = ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i
KAILASH_DF = Kailash::DataFlow.new(config)
```

## MUST NOT Rules

### 1. No Unbounded Connection Creation

MUST NOT create database connections in loops or recursive functions without pool limits.

```ruby
# WRONG -- connection per iteration
user_ids.each do |user_id|
  conn = PG.connect(ENV.fetch("DATABASE_URL"))
  result = conn.exec_params("SELECT * FROM users WHERE id = $1", [user_id])
  conn.close
end

# RIGHT -- use pool, or batch query
POOL.with do |conn|
  result = conn.exec_params(
    "SELECT * FROM users WHERE id = ANY($1)", ["{#{user_ids.join(",")}}"]
  )
end
```

### 2. No Pool Size From User Input

MUST NOT allow pool size to be set from user-controlled input (API parameters, form fields).

### 3. No Separate ConnectionManagers Per Store

MUST NOT create a new ConnectionManager/DataFlow instance for each store. All stores within a worker MUST share the same pool.

## Cross-References

- `rules/deployment.md` -- Production deployment checklist
- `rules/patterns.md` -- DataFlow framework patterns
- `skills/project/pool-safety.md` -- Deployment verification skill
