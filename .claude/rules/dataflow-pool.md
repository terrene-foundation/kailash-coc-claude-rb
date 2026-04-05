---
paths:
  - "lib/**/dataflow/**"
---

# DataFlow Pool Configuration Rules

### 1. Single Source of Truth for Pool Size

Pool size MUST be resolved through `Kailash::DataFlow::Config#pool_size`. No hardcoded defaults elsewhere.

```ruby
# ✅
pool_size = config.pool_size(environment)

# ❌ Competing defaults
pool_size = options.fetch(:pool_size, 10)
pool_size = ENV.fetch("DATAFLOW_POOL_SIZE", "10").to_i
```

**Why:** Five competing defaults (10, 20, 25, 30, `Etc.nprocessors * 4`) caused the pool exhaustion crisis.

### 2. Validate Pool Config at Startup

**Why:** Without startup validation, misconfigured pool sizes silently exhaust `max_connections` under load, causing cryptic `PG::ConnectionBad` errors hours after deployment.

When connecting to PostgreSQL, `validate_pool_config` logs whether pool will exhaust `max_connections`. Runs in `Kailash::DataFlow.new` automatically.

### 3. No Deceptive Configuration

Config flags MUST have backing implementation. A flag set to `true` with no consumer is a stub (`zero-tolerance.md` Rule 2).

**Why:** A config flag with no backing code gives operators false confidence that a feature is active, masking the gap until a production incident reveals it was never wired up.

### 4. Bounded max_overflow

**Why:** Unbounded overflow defeats pool sizing math entirely -- a `pool_size * 2` overflow triples the connection footprint and exhausts `max_connections` during traffic spikes.

```ruby
# ✅
max_overflow = [2, pool_size / 2].max

# ❌ Triples connection footprint
max_overflow = pool_size * 2
```

### 5. No Orphan Runtimes

Subsystem classes MUST accept optional `runtime` parameter. All MUST implement `close` calling `@runtime.release`.

```ruby
# ✅
class SubsystemClass
  def initialize(runtime: nil)
    if runtime
      @runtime = runtime.acquire
      @owns_runtime = false
    else
      @runtime = Kailash::LocalRuntime.new
      @owns_runtime = true
    end
  end

  def close
    return unless @runtime
    @runtime.release
    @runtime = nil
  end
end

# ❌ Orphan — no close, no sharing
class SubsystemClass
  def initialize
    @runtime = Kailash::LocalRuntime.new
  end
end
```

**Why:** Five independent runtimes per DataFlow instance consumed 28-64 connections each.

## MUST NOT

- No new pool size defaults — consolidate with existing parameters before adding
  **Why:** Each new default creates another competing pool size source, recreating the exact five-default crisis that caused the original pool exhaustion bug.
