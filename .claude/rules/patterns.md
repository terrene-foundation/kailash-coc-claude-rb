---
paths:
  - "**/*.rb"
  - "**/Gemfile"
  - "**/Rakefile"
  - "**/*.gemspec"
---

# Ruby SDK Patterns

## Scope

These rules apply when writing Ruby application code using the Kailash SDK.

## MUST Rules

### 1. Block-Based Resource Management

MUST use block form for Runtime to ensure cleanup:

```ruby
# DO: Block form — automatic cleanup
Kailash::Runtime.open(registry) do |rt|
  result = rt.execute(wf, params)
end

# DO NOT: Manual open without ensure
rt = Kailash::Runtime.new(registry)
result = rt.execute(wf, params)
# If exception here, runtime leaks
```

**Why**: The Rust extension allocates native memory. Block form guarantees cleanup via ensure.

### 2. Close Native Objects

MUST close Registry and Workflow when done:

```ruby
# DO:
begin
  registry = Kailash::Registry.new
  wf = builder.build(registry)
  # ... use wf ...
ensure
  wf&.close
  registry&.close
end

# DO NOT: Let GC handle it
registry = Kailash::Registry.new
# ... registry goes out of scope without close
```

**Why**: Ruby's GC does not deterministically call Rust destructors. Explicit close prevents resource leaks.

### 3. Hash Parameters, Not Keyword Arguments

MUST pass parameters as Hash, not keyword arguments:

```ruby
# DO:
builder.add_node("LogNode", "log", { "message" => "hello" })
result = rt.execute(wf, { "input" => value })

# DO NOT:
builder.add_node("LogNode", "log", message: "hello")
```

**Why**: The Magnus FFI bridge expects `Hash<String, Value>`. Symbol keys are not automatically converted.

### 4. String Keys in Hashes

MUST use string keys, not symbols:

```ruby
# DO:
{ "url" => "https://...", "method" => "GET" }

# DO NOT:
{ url: "https://...", method: "GET" }
```

**Why**: The Rust side deserializes via `serde_json`. Symbol keys produce `":"url"` which fails deserialization.

## MUST NOT Rules

### 1. No Mutex Wrapping of Kailash Objects

MUST NOT wrap Kailash objects in Ruby Mutex for thread safety:

```ruby
# DO NOT:
mutex = Mutex.new
mutex.synchronize { rt.execute(wf, params) }
```

**Why**: The native extension already handles thread safety. Ruby Mutex + Rust internal locks = deadlock risk.

### 2. No Fork After Registry Creation

MUST NOT fork the process after creating a Registry:

```ruby
# DO NOT:
registry = Kailash::Registry.new
fork do
  # registry is now in undefined state
end
```

**Why**: Rust's internal state (thread pools, Arc references) does not survive fork. Create Registry after fork.
