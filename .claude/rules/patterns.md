---
paths:
  - "**/*.rb"
  - "**/Gemfile"
  - "**/Rakefile"
  - "**/*.gemspec"
---

# Ruby SDK Patterns

### 1. Block-Based Resource Management

MUST use block form for Runtime to ensure cleanup:

```ruby
# DO:
Kailash::Runtime.open(registry) do |rt|
  result = rt.execute(wf, params)
end

# DO NOT:
rt = Kailash::Runtime.new(registry)
result = rt.execute(wf, params)
# If exception here, runtime leaks
```

**Why:** The Rust extension allocates native memory. Block form guarantees cleanup via ensure.

### 2. Close Native Objects

MUST close Registry and Workflow when done:

```ruby
begin
  registry = Kailash::Registry.new
  wf = builder.build(registry)
  # ... use wf ...
ensure
  wf&.close
  registry&.close
end
```

**Why:** Ruby's GC does not deterministically call Rust destructors.

### 3. Hash Parameters, Not Keyword Arguments

```ruby
# DO:
builder.add_node("LogNode", "log", { "message" => "hello" })

# DO NOT:
builder.add_node("LogNode", "log", message: "hello")
```

**Why:** The Magnus FFI bridge expects `Hash<String, Value>`. Symbol keys are not automatically converted.

### 4. String Keys in Hashes

```ruby
# DO:
{ "url" => "https://...", "method" => "GET" }

# DO NOT:
{ url: "https://...", method: "GET" }
```

**Why:** Rust side deserializes via `serde_json`. Symbol keys produce `":"url"` which fails.

## Framework-Specific Rules

### DataFlow Express (Default for CRUD)

```ruby
# DO: Express API for simple CRUD
result = db.express.create("User", { "name" => "Alice" })
user = db.express.read("User", result["id"].to_s)
users = db.express.list("User", { "active" => true }, limit: 10)
db.express.update("User", result["id"].to_s, { "name" => "Bob" })

# DO NOT: Workflow for single-record CRUD
builder = Kailash::WorkflowBuilder.new
builder.add_node("UserCreateNode", "create", { "name" => "Alice" })
```

### DataFlow (Multi-Step)

```ruby
# Primary key MUST be named 'id'
# Use FLAT params for CreateNode, filter + fields for UpdateNode
builder.add_node("UpdateUser", "update", {
  "filter" => { "id" => 1 },
  "fields" => { "name" => "new_name" }
})
```

### Kaizen

```ruby
delegate = Kailash::Kaizen::Delegate.new(model: ENV["LLM_MODEL"])
delegate.run("Analyze this data") { |event| puts event }
```

## MUST NOT

### 1. No Mutex Wrapping of Kailash Objects

The native extension handles thread safety. Ruby Mutex + Rust internal locks = deadlock risk.

**Why:** The Rust extension already holds internal locks; wrapping with a Ruby `Mutex` creates nested lock acquisition that deadlocks under concurrent access.

### 2. No Fork After Registry Creation

Rust's internal state (thread pools, Arc references) does not survive fork. Create Registry after fork.

**Why:** `fork` copies the Ruby heap but not Rust thread pools or Arc references, leaving the child process with dangling pointers that segfault on first use.
