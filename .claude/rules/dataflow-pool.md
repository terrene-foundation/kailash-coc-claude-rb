---
paths:
  - "lib/**/dataflow/**"
---

# DataFlow Pool Configuration Rules

## Scope

These rules apply when working with DataFlow code in the Ruby SDK.

## MUST Rules

### 1. Single Source of Truth for Pool Size

Pool size MUST be resolved through exactly one code path: `Kailash::DataFlow::Config#pool_size`.

No hardcoded pool size defaults outside this method. Any code that needs a pool size MUST call the config object's method. If the config object does not have a value, that is a config bug -- not a reason to invent a local default.

**Why**: Five competing defaults (10, 20, 25, 30, `Etc.nprocessors * 4`) caused the pool exhaustion crisis. Every new default creates convention drift.

**How to apply**: Before adding `pool_size: N` anywhere, check if the config object's pool size is being used. If not, wire it up rather than hardcoding.

### 2. No Hardcoded Numeric Pool Defaults

MUST NOT add `pool_size: N` as a default in constructors, env var fallbacks, or adapter base classes. All defaults flow through `Kailash::DataFlow::Config#pool_size`.

```ruby
# DO:
pool_size = config.pool_size(environment)

# DO NOT:
pool_size = options.fetch(:pool_size, 10)  # Competing default!
pool_size = ENV.fetch("DATAFLOW_POOL_SIZE", "10").to_i  # Ghost code!
```

### 3. Validate Pool Config at Startup

When connecting to PostgreSQL, MUST call `validate_pool_config` to log whether the configured pool will exhaust `max_connections`. This runs in `Kailash::DataFlow.new` automatically.

### 4. No Deceptive Configuration

Config fields that suggest a feature exists MUST have a backing implementation. A config flag set to `true` by default with no consumer is functionally a stub and violates `no-stubs.md` Rule 4.

**Why**: `MonitoringConfig.alert_on_connection_exhaustion = true` with no backing code led users to believe they had exhaustion protection when they didn't.

### 5. Bounded max_overflow

MUST NOT compute `max_overflow = pool_size * 2`. This triples the connection footprint. Use `[2, pool_size / 2].max` instead.

```ruby
# DO:
max_overflow = [2, pool_size / 2].max

# DO NOT:
max_overflow = pool_size * 2  # Triples connection footprint!
```

### 6. No Orphan Runtimes

DataFlow subsystem classes MUST accept an optional `runtime` parameter. If provided, call `runtime.acquire` and store. If `nil`, create own runtime. All classes MUST implement `close` that calls `@runtime.release`.

```ruby
# DO:
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

# DO NOT:
class SubsystemClass
  def initialize
    @runtime = Kailash::LocalRuntime.new  # Orphan -- no close, no sharing
  end
end
```

**Why**: Five independent runtimes per DataFlow instance caused the connection pool exhaustion crisis. Each runtime opens 7-16 connections; without sharing, a single `Kailash::DataFlow.new(auto_migrate: true)` consumed 28-64 connections.

**Enforced by**: `validate-workflow.js` emits WARNING on unmanaged `LocalRuntime.new` construction.

## MUST NOT Rules

### 1. No New Pool Size Defaults

When adding a new config parameter, search for existing parameters with similar names or purposes. Consolidate before adding. The pool default drift incident (five competing defaults) is the canonical example of what happens when this rule is violated.

## Cross-References

- `01-analysis/01-codebase-defects.md` -- DEFECT-A (five competing defaults)
- `01-analysis/04-cross-sdk-alignment.md` -- kailash-rs uses same auto-scaling formula
- `rules/connection-pool.md` -- Connection pool safety rules (application-level)
