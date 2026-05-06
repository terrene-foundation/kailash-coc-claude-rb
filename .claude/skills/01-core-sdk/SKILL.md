---
name: core-sdk
description: "Kailash Ruby SDK — gem setup, workflows, nodes, runtime, Magnus FFI bindings."
---

# Kailash Ruby SDK

High-performance workflow automation SDK powered by Rust via Magnus FFI bindings.

## Install

```bash
gem install kailash
```

Or in Gemfile:

```ruby
gem "kailash"
```

## Quick Start

```ruby
require "kailash"

# 1. Create a registry and workflow
registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new

# 2. Add nodes
builder.add_node("NoOpNode", "start", {})
builder.add_node("LogNode", "log", { "message" => "Hello from Ruby!" })
builder.connect("start", "log")

# 3. Build and execute
workflow = builder.build(registry)
Kailash::Runtime.open(registry) do |rt|
  result = rt.execute(workflow, {})
  puts result.results
end
workflow.close
```

## Core Types

| Type                       | Purpose                                     |
| -------------------------- | ------------------------------------------- |
| `Kailash::Registry`        | Node type registry (must outlive workflows) |
| `Kailash::WorkflowBuilder` | DAG builder — add_node, connect, build      |
| `Kailash::Runtime`         | Workflow executor — open/close lifecycle    |
| `Kailash::Workflow`        | Compiled workflow (from builder.build)      |

## Lifecycle Pattern

```ruby
registry = Kailash::Registry.new

begin
  builder = Kailash::WorkflowBuilder.new
  builder.add_node("NoOpNode", "n1", {})
  wf = builder.build(registry)

  Kailash::Runtime.open(registry) do |rt|
    result = rt.execute(wf, { "input" => "value" })
    # result.results => Hash of node outputs
    # result.run_id => String UUID
  end
ensure
  wf&.close
  registry&.close
end
```

**Critical**: Always close Registry and Workflow. Use `Runtime.open` block form for automatic cleanup.

## Parameter Passing

```ruby
# Input parameters to workflow
result = rt.execute(wf, { "user_id" => 42, "name" => "Alice" })

# Node parameters (set at build time)
builder.add_node("HttpRequestNode", "fetch", {
  "url" => "https://api.example.com/users",
  "method" => "GET"
})
```

## Error Handling

```ruby
begin
  wf = builder.build(registry)
rescue Kailash::BuildError => e
  puts "Workflow validation failed: #{e.message}"
end

begin
  result = rt.execute(wf, {})
rescue Kailash::ExecutionError => e
  puts "Execution failed: #{e.message}"
end
```

## Available Node Types

139 node types across 25+ categories. Use `registry.list_types` to enumerate.

Key categories: AI (9), HTTP (8), SQL (3), File (7), Auth (12), Security (12), Monitoring (10), Transaction (5), Enterprise (8), Code (4), RAG (7), Cache (3), Streaming (3).

## Thread Safety

All Kailash types are `Send + Sync` in Rust. Ruby's GVL applies — use `Ractor` or process-based parallelism for true concurrency. The native extension releases the GVL during Rust execution.

## Sub-Skills

- `ruby-quickstart.md` — Step-by-step getting started
- `ruby-available-nodes.md` — Full node reference
- `ruby-custom-nodes.md` — Custom node development
- `ruby-common-mistakes.md` — Pitfalls and solutions
- `ruby-cheatsheet.md` — Quick reference card
