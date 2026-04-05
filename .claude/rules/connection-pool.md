---
paths:
  - "**/*.rb"
---

# Connection Pool Safety Rules

### 1. Never Use Default Pool Size in Production

Set `DATAFLOW_MAX_CONNECTIONS` env var. Default (25/worker) exhausts PostgreSQL on small instances.

**Why:** Puma/Unicorn fork workers that each inherit the default pool size, so 4 workers x 25 connections = 100 connections from a single app, exhausting small-instance `max_connections`.

**Formula**: `pool_size = postgres_max_connections / num_workers * 0.7`

| Instance        | `max_connections` | Workers | `DATAFLOW_MAX_CONNECTIONS` |
| --------------- | ----------------- | ------- | -------------------------- |
| t2.micro        | 87                | 2       | 30                         |
| t2.small/medium | 150               | 2       | 50                         |
| t3.medium       | 150               | 4       | 25                         |
| r5.large        | 1000              | 4       | 175                        |

```ruby
# ❌ relies on default pool size
config = Kailash::DataFlow::Config.new("postgresql://...")
df = Kailash::DataFlow.new(config)

# ✅ explicit pool size from environment
config = Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL"))
config.max_connections = ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i
df = Kailash::DataFlow.new(config)
```

### 2. Never Query DB Per-Request in Middleware

**Why:** Rack middleware runs before the application checks out its own connection, so a DB query in middleware doubles connection checkout per request and rapidly exhausts the pool.

Creates N+1 connection usage, rapidly exhausting pool.

```ruby
# ❌ DB query on EVERY request
class AuthMiddleware
  def call(env)
    Kailash::Registry.open do |registry|
      builder = Kailash::WorkflowBuilder.new
      builder.add_node("ReadUser", "read", { "token" => token })
      # Pool checkout per request!
    end
  end
end

# ✅ JWT claims, no DB hit
class AuthMiddleware
  def call(env)
    token = env["HTTP_AUTHORIZATION"]&.delete_prefix("Bearer ")
    claims = JWT.decode(token, ENV.fetch("JWT_SECRET"), true, algorithm: "HS256").first
    env["user_id"] = claims["sub"]
    @app.call(env)
  end
end

# ✅ In-memory cache with TTL
SESSION_CACHE = LruRedux::TTL::ThreadSafeCache.new(1000, 300)
```

### 3. Health Checks Must Not Use Application Pool

**Why:** Load balancers hit health endpoints every few seconds; if each check borrows from the application pool, health checks starve real requests under load.

Use lightweight `SELECT 1` with dedicated connection, never a full DataFlow workflow.

```ruby
# ✅
get "/health" do
  ActiveRecord::Base.connection.execute("SELECT 1")
  { status: "ok" }.to_json
rescue StandardError => e
  status 503
  { status: "error", detail: e.message }.to_json
end
```

### 4. Verify Pool Math at Deployment

**Why:** Exceeding `max_connections` causes immediate `PG::ConnectionBad` errors across all workers, taking the entire application offline.

```
DATAFLOW_MAX_CONNECTIONS × num_workers ≤ postgres_max_connections × 0.7
```

### 5. Connection Timeout Must Be Set

Without timeout, requests queue indefinitely when pool exhausted → cascading failures.

**Why:** Sequel and ActiveRecord default to no timeout, so an exhausted pool blocks all threads indefinitely, turning a pool shortage into a full application hang.

```ruby
config.connection_timeout = 5  # seconds
```

### 6. Workers Must Share Pool

**Why:** Each `Kailash::DataFlow.new` call creates a fresh Sequel connection pool, so instantiating per-request multiplies connections by request volume.

Application-level singleton. MUST NOT create new pool per request.

```ruby
# ❌ new pool per request
get "/users" do
  df = Kailash::DataFlow.new(config)  # New pool!
end

# ✅ shared at application level (Sinatra)
configure do
  set :df, Kailash::DataFlow.new(
    Kailash::DataFlow::Config.new(ENV.fetch("DATABASE_URL")).tap { |c|
      c.max_connections = ENV.fetch("DATAFLOW_MAX_CONNECTIONS", "10").to_i
    }
  )
end
```

## MUST NOT

- No unbounded connection creation in loops — use pool or batch queries
  **Why:** A loop opening connections without checkout/checkin exhausts PostgreSQL `max_connections` in seconds, crashing all clients on the database.
- No pool size from user input (API params, form fields)
  **Why:** A malicious request setting `pool_size=10000` can exhaust database connections or allocate unbounded memory for pool metadata.
- No separate connection pools per store — share via application singleton
  **Why:** Separate pools per store multiply the connection footprint by the number of stores, defeating pool sizing math and exhausting `max_connections`.
